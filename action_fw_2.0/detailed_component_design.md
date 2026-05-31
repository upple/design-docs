# Tizen Action Framework 2.0: 산출물별 상세 설계 사양서

> **목적**: Tizen Action Framework 2.0에 정의된 6대 개발 산출물의 클래스 구조, 데이터베이스 쿼리, API 규격, 컴파일러 로직을 정밀하게 보완하여 플랫폼에 직접 개발/이식이 가능한 수준의 프로덕션 상세 설계 사양을 제공합니다.  
> **대상**: Tizen 플랫폼 네이티브 개발자, C# SDK 개발자, AI 에이전트 인프라 설계자

---

## 1. [산출물 1] IPC 인터페이스 레이어 (TIDL)

플랫폼 데몬 서비스(`tizen-action-service`)와 클라이언트 라이브러리(`api_stub.cc`) 간 RPC 통신을 처리하는 TIDL 프로토콜 정의입니다. 2-Stage 비동기 디스패치 및 카테고리 기반 API가 추가 확장되었습니다.

### 1.1 `tidl/tizen_action_datatype.tidl` (데이터 타입 확장 정의)
```
protocol 2

struct AppImpl {
    string pkgid;             // 패키지 ID (예: "com.netflix.tizen")
    string appid;             // 실제 실행 대상 appid (예: "com.netflix.free")
    string label;             // 사용자 노출용 앱 이름 (예: "Netflix")
    bool   is_installed;      // 현재 기기 설치 여부
    string subscription_hint; // "subscribed" | "not_subscribed" | "unknown"
    int    usage_count;       // 최근 30일 누적 사용 횟수 (우선순위 가중치)
}

struct VectorDbResult {
    string action_id;
    float score;
}

struct ActionEvent {
    enum Type {
        Install,
        Uninstall,
        Update
    }
}
```

### 1.2 `tidl/tizen_actions.tidl` (서비스 인터페이스 최종 선언)
```
protocol 2
import <tizen_action_datatype.tidl>

interface ActionService {
    // ── [기존 API 유지 - 하위 호환성 확보] ──────────────────
    array<string> ListActions();
    string GetAction(string action_id);
    array<VectorDbResult> GetActionId(string user_description, int top_k, float search_threshold);
    int Execute(string model);
    
    // ── [신규 API - 카테고리 표준 액션 확장] ──────────────────
    array<string> ListCategories();
    array<string> ListCategoryActions(string category);
    array<AppImpl> ListAppsForAction(string action_name);
    
    // 2-Stage 비동기 실행: 1단계 ACK 즉시 리턴, 2단계 실행 완료는 delegate 콜백으로 수신
    int ExecuteWithAppId(string model, string appid, int execution_id);

    // ── [비동기 콜백 등록 정의] ─────────────────────────────
    void ActionReplyCb(int execution_id, string result_json_rpc) delegate;
    int RegisterActionReplyCb(ActionReplyCb cb);
    bool UnregisterActionReplyCb(int id);
    
    void ActionEventCb(string action_id, ActionEvent.Type event) delegate;
    int RegisterActionEventCb(ActionEventCb cb);
    bool UnregisterActionEventCb(int id);
}
```

---

## 2. [산출물 2] 플랫폼 C++ 데몬 서비스 (`tizen-action-service`)

### 2.1 인메모리 웜 캐시 설계 (`src/action/action_cache.hh`)
동적 액션 매핑을 탐색할 때 SQLite DB 디스크 I/O를 우회하여 5ms 이하 극초지연 조회를 실현하기 위한 메모리 구조체 및 캐시 클래스 설계입니다.

```cpp
#pragma once

#include <string>
#include <vector>
#include <unordered_map>
#include <shared_mutex>
#include <memory>

namespace tizen {
namespace action {

struct CacheEntry {
  std::string pkgid;
  std::string appid;
  std::string category;
  std::string label;
  std::string raw_schema;
  bool is_installed;
  std::string subscription_hint;
  int usage_count;
};

class ActionCache {
 public:
  static ActionCache& GetInstance() {
    static ActionCache instance;
    return instance;
  }

  // SQLite DB 초기 데이터 웜업 로딩
  bool WarmUp(const std::string& db_path);

  // 패키지 설치/업데이트 시 부분 갱신 (Dirty-Flag 무효화 전략)
  void UpdatePackage(const std::string& pkgid, const std::vector<CacheEntry>& entries);

  // 패키지 삭제 시 캐시 제거
  void RemovePackage(const std::string& pkgid);

  // 런타임 액션 기반 구현 앱 목록 조회 (Lock-free Shared Read Lock)
  std::vector<CacheEntry> LookupAppsForAction(const std::string& action_name) const;

  // 힙 메모리 자원 상태 조회 (Pre-Mortem 방어 모니터링용)
  size_t GetTotalHeapFootprintBytes() const;

 private:
  ActionCache() = default;
  ~ActionCache() = default;

  // Key: action_name, Value: 구현된 앱 정보 벡터
  std::unordered_map<std::string, std::vector<CacheEntry>> cache_map_;
  mutable std::shared_mutex cache_mutex_; // Reader-Writer Lock 격리
  bool is_warmed_up_ = false;
  size_t allocated_bytes_ = 0;
};

} // namespace action
} // namespace tizen
```

### 2.2 인메모리 웜 캐시 구현 (`src/action/action_cache.cc`)
```cpp
#include "action_cache.hh"
#include "action/sqlite_db.hh"
#include "common/utils/logging.hh"

namespace tizen {
namespace action {

bool ActionCache::WarmUp(const std::string& db_path) {
  std::unique_lock<std::shared_mutex> lock(cache_mutex_);
  if (is_warmed_up_) return true;

  auto db = std::make_unique<action::SqliteDb>(db_path);
  auto raw_entries = db->GetAllCategoryActionImpls();

  cache_map_.clear();
  allocated_bytes_ = 0;

  for (auto& entry : raw_entries) {
    cache_map_[entry.category + "." + entry.pkgid].push_back(entry);
    allocated_bytes_ += sizeof(CacheEntry) + 
                        entry.pkgid.capacity() + 
                        entry.appid.capacity() + 
                        entry.category.capacity() + 
                        entry.raw_schema.capacity();
  }

  is_warmed_up_ = true;
  LOG(INFO) << "ActionCache Warm-up complete. Allocated memory: " 
            << (allocated_bytes_ / 1024) << " KB";
  return true;
}

std::vector<CacheEntry> ActionCache::LookupAppsForAction(const std::string& action_name) const {
  std::shared_lock<std::shared_mutex> lock(cache_mutex_);
  
  auto it = cache_map_.find(action_name);
  if (it == cache_map_.end()) {
    return std::vector<CacheEntry>();
  }
  return it->second;
}

void ActionCache::UpdatePackage(const std::string& pkgid, const std::vector<CacheEntry>& entries) {
  std::unique_lock<std::shared_mutex> lock(cache_mutex_);
  
  // 기존 패키지 정보 선제 삭제
  for (auto& pair : cache_map_) {
    auto& vec = pair.second;
    vec.erase(std::remove_if(vec.begin(), vec.end(),
                             [&pkgid](const CacheEntry& e) { return e.pkgid == pkgid; }),
              vec.end());
  }

  // 신규 정보 추가
  for (const auto& entry : entries) {
    cache_map_[entry.category].push_back(entry);
  }
  LOG(DEBUG) << "ActionCache dynamically updated for package: " << pkgid;
}

void ActionCache::RemovePackage(const std::string& pkgid) {
  std::unique_lock<std::shared_mutex> lock(cache_mutex_);
  for (auto& pair : cache_map_) {
    auto& vec = pair.second;
    vec.erase(std::remove_if(vec.begin(), vec.end(),
                             [&pkgid](const CacheEntry& e) { return e.pkgid == pkgid; }),
              vec.end());
  }
  LOG(DEBUG) << "ActionCache purged for package: " << pkgid;
}

size_t ActionCache::GetTotalHeapFootprintBytes() const {
  std::shared_lock<std::shared_mutex> lock(cache_mutex_);
  return allocated_bytes_;
}

} // namespace action
} // namespace tizen
```

### 2.3 `CategoryActionRegistry` 설계 (`src/common/category_action_registry.hh`)
플랫폼 정의 표준 스키마 딕셔너리 파일을 디바이스 부팅 시 파싱하고, 3rd Party 앱 설치 시점 스키마 정합성 검사를 관장하는 핵심 정적 레지스트리 클래스입니다.

```cpp
#pragma once

#include <string>
#include <vector>
#include <unordered_map>

namespace tizen {
namespace common {

struct StandardActionSchema {
  std::string name;
  std::string description;
  std::string input_schema_json;
  std::string output_schema_json;
  std::vector<std::string> required_fields;
};

class CategoryActionRegistry {
 public:
  static CategoryActionRegistry& GetInstance() {
    static CategoryActionRegistry instance;
    return instance;
  }

  // platform-category-actions/*.json 스키마 디렉터리 일괄 로드
  bool LoadPlatformStandards(const std::string& schema_dir_path);

  // 카테고리 존재 여부 검사
  bool IsStandardCategory(const std::string& category) const;

  // 특정 카테고리의 특정 액션명이 표준 규격인지 검사
  bool IsStandardAction(const std::string& category, const std::string& action_name) const;

  // 표준 입력/출력 JSON 스키마 템플릿 획득
  bool GetStandardSchema(const std::string& category, const std::string& action_name, 
                          StandardActionSchema& out_schema) const;

 private:
  CategoryActionRegistry() = default;
  
  // Key: category, Value: (Key: action_name, Value: 표준 스펙)
  std::unordered_map<std::string, std::unordered_map<std::string, StandardActionSchema>> registry_;
};

} // namespace common
} // namespace tizen
```

---

## 3. [산출물 3] SQLite 데이터베이스 연동부 (`SqliteDb`)

### 3.1 DDL (Data Definition Language) 명세
하위 호환성 유지를 위해 기존 `action` 및 `privilege` 테이블을 보존하고, 신규 2개 테이블을 정규화하여 설계합니다.

```sql
PRAGMA foreign_keys = ON;

-- 1. 플랫폼이 선언한 표준 카테고리 정의 테이블
CREATE TABLE IF NOT EXISTS category_action (
  category    TEXT NOT NULL,
  action_name TEXT NOT NULL,
  json_schema TEXT NOT NULL,   -- 표준 inputSchema 및 outputSchema를 포함한 압축 스펙
  version     INTEGER NOT NULL DEFAULT 1,
  PRIMARY KEY (category, action_name)
);

-- 2. 3rd Party 앱이 구현을 보장하고 선언한 표준 카테고리 맵
-- 하나의 패키지(pkgid) 내 여러 appid(무료/프리미엄 버전) 구분을 위해 appid 복합키 지정
CREATE TABLE IF NOT EXISTS category_action_impl (
  pkgid             TEXT NOT NULL,
  action_name       TEXT NOT NULL,
  category          TEXT NOT NULL,
  appid             TEXT NOT NULL,
  label             TEXT,            -- 사용자 노출명 
  subscription_hint TEXT NOT NULL DEFAULT 'unknown', -- 'subscribed' | 'not_subscribed' | 'unknown'
  usage_count       INTEGER NOT NULL DEFAULT 0,
  PRIMARY KEY (pkgid, action_name, appid),
  FOREIGN KEY (category, action_name) 
    REFERENCES category_action(category, action_name) ON DELETE CASCADE
);
```

### 3.2 `src/action/sqlite_db.cc` 웜업 및 조회 쿼리 실무 구현
```cpp
#include "action/sqlite_db.hh"
#include <sqlite3.h>
#include "common/utils/logging.hh"

namespace tizen {
namespace action {

// Pre-Mortem 방어: SQLite WAL(Write-Ahead Logging) 모드를 강제 활성화하고 DB 커넥션 오픈
sqlite3* OpenDatabaseWithWAL(const std::string& db_path) {
  sqlite3* db = nullptr;
  int rc = sqlite3_open_v2(db_path.c_str(), &db, 
                           SQLITE_OPEN_READWRITE | SQLITE_OPEN_CREATE | SQLITE_OPEN_FULLMUTEX, 
                           nullptr);
  if (rc != SQLITE_OK) {
    LOG(ERROR) << "Failed to open database: " << sqlite3_errmsg(db);
    sqlite3_close(db);
    return nullptr;
  }

  // WAL 모드 PRAGMA 활성화
  char* zErrMsg = nullptr;
  rc = sqlite3_exec(db, "PRAGMA journal_mode=WAL;", nullptr, nullptr, &zErrMsg);
  if (rc != SQLITE_OK) {
    LOG(WARNING) << "Failed to set WAL mode: " << zErrMsg;
    sqlite3_free(zErrMsg);
  }
  return db;
}

std::vector<CacheEntry> SqliteDb::GetAllCategoryActionImpls() {
  std::vector<CacheEntry> results;
  sqlite3* db = OpenDatabaseWithWAL(db_path_);
  if (!db) return results;

  const char* query = "SELECT pkgid, appid, category, action_name, label, subscription_hint, usage_count "
                      "FROM category_action_impl;";
  sqlite3_stmt* stmt = nullptr;

  if (sqlite3_prepare_v2(db, query, -1, &stmt, nullptr) == SQLITE_OK) {
    while (sqlite3_step(stmt) == SQLITE_ROW) {
      CacheEntry entry;
      entry.pkgid = reinterpret_cast<const char*>(sqlite3_column_text(stmt, 0));
      entry.appid = reinterpret_cast<const char*>(sqlite3_column_text(stmt, 1));
      entry.category = reinterpret_cast<const char*>(sqlite3_column_text(stmt, 2));
      entry.label = reinterpret_cast<const char*>(sqlite3_column_text(stmt, 4));
      entry.subscription_hint = reinterpret_cast<const char*>(sqlite3_column_text(stmt, 5));
      entry.usage_count = sqlite3_column_int(stmt, 6);
      results.push_back(entry);
    }
  }
  sqlite3_finalize(stmt);
  sqlite3_close(db);
  return results;
}

} // namespace action
} // namespace tizen
```

---

## 4. [산출물 4] 패키지 매니저 메타데이터 플러그인 (`Metadata Plugin`)

앱 설치/업데이트/삭제 수명 주기 이벤트 발생 시, 앱의 선언 파일(`res/action.json`)에 명시된 카테고리 액션이 플랫폼 표준에 적합한지 유효성을 검증하고 에러 시 설치를 단호하게 거부하는 핵심 파서 로직입니다.

### 4.1 `src/pkgmgr_plugin_parser/json_action_schema_parser.cc` (표준 적합성 엄격 검증 로직)
```cpp
#include <rapidjson/document.h>
#include <rapidjson/schema.h>
#include "common/category_action_registry.hh"
#include "common/utils/logging.hh"

namespace tizen {
namespace pkgmgr_plugin {

enum class SchemaValidationResult {
  VALID,
  INVALID_JSON,
  NON_STANDARD_CATEGORY,
  NON_STANDARD_ACTION,
  REQUIRED_PARAMETER_MISSING,
  PARAMETER_TYPE_MISMATCH
};

SchemaValidationResult ValidateActionCompatibility(const std::string& json_str) {
  rapidjson::Document doc;
  if (doc.Parse(json_str.c_str()).HasParseError()) {
    return SchemaValidationResult::INVALID_JSON;
  }

  // 1. 카테고리 필드 존재 유무 검증
  if (!doc.HasMember("category") || !doc["category"].IsString()) {
    // 카테고리가 없는 경우 임의 생성 액션으로 취급하여 밸리데이션 패스 (기존 테이블 삽입)
    return SchemaValidationResult::VALID;
  }

  std::string category = doc["category"].GetString();
  std::string action_name = doc["name"].GetString();

  auto& registry = common::CategoryActionRegistry::GetInstance();

  // 2. 플랫폼 선언 표준 카테고리인지 1차 필터링
  if (!registry.IsStandardCategory(category)) {
    LOG(ERROR) << "Installation Blocked: non-standard category detected -> " << category;
    return SchemaValidationResult::NON_STANDARD_CATEGORY;
  }

  // 3. 해당 카테고리에 정의된 표준 액션 이름인지 2차 필터링
  if (!registry.IsStandardAction(category, action_name)) {
    LOG(ERROR) << "Installation Blocked: non-standard action name detected -> " << action_name;
    return SchemaValidationResult::NON_STANDARD_ACTION;
  }

  // 4. 표준 스키마 템플릿과 파라미터 규격 대조 (일치성 검사)
  common::StandardActionSchema standard;
  registry.GetStandardSchema(category, action_name, standard);

  rapidjson::Document std_input;
  std_input.Parse(standard.input_schema_json.c_str());

  if (doc.HasMember("inputSchema") && doc["inputSchema"].IsObject()) {
    auto& input_properties = doc["inputSchema"]["properties"];
    auto& std_properties = std_input["properties"];

    // 표준 스키마의 필수 파라미터가 구현 JSON에도 정의되어 있는지 타입 매칭 검증
    for (auto it = std_properties.MemberBegin(); it != std_properties.MemberEnd(); ++it) {
      std::string param_name = it->name.GetString();
      std::string expected_type = it->value["type"].GetString();

      if (!input_properties.HasMember(param_name.c_str())) {
        LOG(ERROR) << "Schema Verification Failed: Missing required parameter -> " << param_name;
        return SchemaValidationResult::REQUIRED_PARAMETER_MISSING;
      }

      std::string actual_type = input_properties[param_name.c_str()]["type"].GetString();
      if (actual_type != expected_type) {
        LOG(ERROR) << "Schema Verification Failed: Parameter type mismatch on " << param_name 
                   << " (Expected: " << expected_type << ", Actual: " << actual_type << ")";
        return SchemaValidationResult::PARAMETER_TYPE_MISMATCH;
      }
    }
  }

  return SchemaValidationResult::VALID;
}

} // namespace pkgmgr_plugin
} // namespace tizen
```

---

## 5. [산출물 5] C# Application SDK 및 Roslyn Source Generator

### 5.1 C# `[TizenAction]` 선언적 개발자 경험 API 설계
```csharp
using System;

namespace Tizen.Applications.Action
{
    [AttributeUsage(AttributeTargets.Method, Inherited = false, AllowMultiple = true)]
    public sealed class TizenActionAttribute : Attribute
    {
        public string Category { get; }
        public string ActionName { get; }
        public string Description { get; set; } = string.Empty;
        public string[] RequiredPrivileges { get; set; } = Array.Empty<string>();

        public TizenActionAttribute(string category, string actionName)
        {
            Category = category;
            ActionName = actionName;
        }
    }

    [AttributeUsage(AttributeTargets.Parameter)]
    public sealed class ActionParamAttribute : Attribute
    {
        public string Description { get; set; } = string.Empty;
        public bool Required { get; set; } = true;
    }
}
```

### 5.2 Roslyn Generator가 컴파일 타임에 생성할 템플릿 아키텍처 모델
컴파일 과정에서 Roslyn 소스 생성기(`TizenActionSourceGenerator.dll`)가 사용자의 코드 어노테이션을 파싱하여 빌드 결과물에 삽입할 디스패치 보일러플레이트 코드의 최종 스펙 구조입니다.

```csharp
// <auto-generated/>
// 컴파일러에 의해 자동 생성된 TizenActionDispatcher.g.cs 코드 예시
using System;
using Tizen.Applications;

namespace Tizen.Applications.Action.Generated
{
    public static class TizenActionDispatcher
    {
        public static void HandleAppControl(AppControlReceivedEventArgs e, object appInstance)
        {
            string operation = e.ReceivedAppControl.Operation;
            if (operation != "http://tizen.org/appcontrol/operation/action") return;

            string actionName = e.ReceivedAppControl.ExtraData.Get<string>("http://tizen.org/appcontrol/extra/action_name");
            string payloadJson = e.ReceivedAppControl.ExtraData.Get<string>("http://tizen.org/appcontrol/extra/payload");

            if (actionName == "play_content")
            {
                // JSON 파라미터 파싱 및 파라미터 타입 자동 변환 매핑
                var parser = new SimpleJsonParser(payloadJson);
                string contentId = parser.GetString("content_id");
                int startTime = parser.GetInt("start_time");

                // 사용자가 구현한 PlayVideo 메서드 다이렉트 런타임 바인딩 호출 (Reflection 회피하여 지연 제거)
                var targetApp = (MyStreamingApp)appInstance;
                var result = targetApp.PlayVideo(contentId, startTime);

                // 2-Stage 비동기 리턴: 결과를 JSON-RPC 형식으로 가공하여 Action Service로 복구 통지
                var reply = new AppControl();
                reply.ExtraData.Add("result", result.ToJson());
                AppControl.SendReply(e.ReceivedAppControl, reply, AppControlReplyResult.Succeeded);
            }
        }
    }
}
```

---

## 6. [산출물 6] 공개 Client C API (`action.h`)

3rd Party 에이전트 개발사나 프레임워크 런타임이 동적으로 카테고리를 탐색하고 특정 패키지를 디스패치할 때 사용하는 표준 Native C API 헤더 정의입니다.

```c
#ifndef _TIZEN_ACTION_H_
#define _TIZEN_ACTION_H_

#include <stdbool.h>

#ifdef __cplusplus
extern "C" {
#endif

typedef struct action_client_s* action_client_h;

typedef enum {
    ACTION_ERROR_NONE = 0,
    ACTION_ERROR_INVALID_PARAMETER = -22,      /* EINVAL */
    ACTION_ERROR_OUT_OF_MEMORY = -12,          /* ENOMEM */
    ACTION_ERROR_IO_ERROR = -5,                /* EIO */
    ACTION_ERROR_PERMISSION_DENIED = -13,      /* EACCES */
    ACTION_ERROR_NOT_SUPPORTED_CATEGORY = -95, /* EOPNOTSUPP */
    ACTION_ERROR_APP_EXECUTION_FAILED = -125   /* ECANCELED */
} action_error_e;

/**
 * @brief Action Client 커넥션을 생성하고 세션을 초기화합니다.
 */
int action_client_create(action_client_h* client);

/**
 * @brief Action Client 커넥션을 안전하게 해제합니다.
 */
int action_client_destroy(action_client_h client);

/**
 * @brief 특정 표준 카테고리를 지원하는 등록 앱 목록을 조회합니다.
 */
int action_client_foreach_apps_for_action(
    action_client_h client,
    const char* action_name,
    bool (*cb)(const char* pkgid, const char* appid, const char* label, void* user_data),
    void* user_data);

/**
 * @brief 비동기 2-Stage 디스패치 기반 표준 카테고리 액션을 실행합니다.
 * @param[in] client       클라이언트 세션 핸들
 * @param[in] model_json   경량화된 스키마 형식의 input 파라미터 JSON
 * @param[in] appid        실행할 대상 3rd Party Target AppID
 * @param[in] callback     실행 완료 시 백그라운드로 수신할 2단계 콜백 함수 포인터
 */
int action_client_execute_with_appid_async(
    action_client_h client,
    const char* model_json,
    const char* appid,
    void (*callback)(int execution_id, const char* result_json, void* user_data),
    void* user_data,
    int* execution_id);

#ifdef __cplusplus
}
#endif

#endif /* _TIZEN_ACTION_H_ */
```
