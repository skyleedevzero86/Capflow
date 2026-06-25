<img width="1373" height="762" alt="image" src="https://github.com/user-attachments/assets/e42ea9ce-ec1e-47db-939f-f7a99325d751" />

<br/>


# Capflow

Capflow는 웹 화면을 캡처하고, 폴더별로 정리하며, 스크린샷에 주석을 추가하고, 단계별 매뉴얼을 제작할 수 있는 Windows 데스크톱 애플리케이션입니다.

## 아키텍처

본 솔루션은 **헥사고날 아키텍처(Hexagonal Architecture, Ports and Adapters)**, **도메인 주도 설계(DDD, Domain-Driven Design)**, 그리고 **SOLID 원칙**을 기반으로 설계되었습니다.

```text
Capflow/
├── src/
│   ├── Capflow.Domain/           # 애그리게이트, 엔티티, 도메인 이벤트, 리포지토리 및 포트 인터페이스
│   ├── Capflow.Application/      # 유스케이스, DTO, Result 패턴, 세션 스코프
│   ├── Capflow.Infrastructure/   # JSON 저장소, 파일 이미지 저장소, 도메인 이벤트 핸들러
│   └── Capflow.Presentation/     # WPF UI, ViewModel, 플랫폼 어댑터(WebView2, 단축키)
├── tests/
│   ├── Capflow.Domain.Tests/
│   └── Capflow.Application.Tests/
└── resources/
    └── icon.png
```

### 계층별 역할

| 계층                 | 역할                                                                         |
| ------------------ | -------------------------------------------------------------------------- |
| **Domain**         | `Project` 애그리게이트, 도메인 이벤트, `IProjectImageStore` 포트                         |
| **Application**    | 단일 책임 유스케이스, `Result<T>`, `IProjectSessionScope`                           |
| **Infrastructure** | `JsonProjectRepository`, `FileProjectImageStore`, `ProjectSaveCoordinator` |
| **Presentation**   | WPF 화면, MVVM ViewModel, 오류 알림을 위한 `IUserNotificationService`               |

### 세션 스코프(Session Scope)

* `IProjectSessionScopeFactory`는 프로젝트 이동 시 프로젝트별 스코프 세션을 생성합니다.
* 홈 화면으로 돌아가거나 다른 프로젝트로 전환할 때 `NavigationService.ActiveScope`가 해제(Dispose)됩니다.
* `ActiveSession`은 캡처 및 편집 유스케이스에서 사용하는 실제 `IProjectSession`을 제공합니다.

### 이미지 저장 방식

스크린샷은 JSON 내부에 포함되지 않고 PNG 파일로 저장됩니다.

```text
%AppData%\Capflow\assets\{projectId}\images\{screenshotId}.png
```

기존 JSON에 `ImageBase64` 형식으로 저장된 프로젝트는 로드 시 자동으로 마이그레이션됩니다.

### 도메인 이벤트

* `Project` 애그리게이트에서 `FolderAddedDomainEvent`, `CaptureAddedDomainEvent`가 발생합니다.
* `ProjectSaveCoordinator`가 데이터를 저장한 후 `IDomainEventDispatcher`를 통해 이벤트를 전달합니다.
* Infrastructure 계층의 감사(Audit) 핸들러는 `ILogger`를 사용하여 이벤트를 기록합니다.

### 유스케이스 (작업별 인터페이스 분리)

| 유스케이스                                                    | 역할                        |
| -------------------------------------------------------- | ------------------------- |
| `IAddCaptureFolderUseCase`                               | 캡처 폴더 생성                  |
| `IRenameCaptureFolderUseCase`                            | 폴더 이름 변경                  |
| `IRemoveCaptureFolderUseCase`                            | 폴더 및 이미지 파일 삭제            |
| `IRemoveCaptureImageUseCase`                             | 개별 스크린샷 삭제                |
| `IListCaptureFoldersUseCase`                             | 폴더 및 이미지 목록 조회            |
| `ICapturePageUseCase`                                    | 이미지 파일 저장 및 캡처 등록         |
| `IEditorQueryUseCase`                                    | 편집기 조회 모델 제공(비동기)         |
| `ISaveEditorFolderUseCase`                               | 편집기 폴더 생성 및 수정            |
| `IRemoveEditorFolderUseCase`                             | 편집기 폴더 삭제                 |
| `IUpdateEditorFolderMetadataUseCase`                     | 경로 및 설명 수정                |
| `IReorderEditorScreenshotsUseCase`                       | 스크린샷 순서 변경                |
| `IAddAnnotationUseCase`                                  | 번호형 주석 추가                 |
| `IRemoveAnnotationUseCase` / `IRestoreAnnotationUseCase` | 실행 취소(Undo) / 다시 실행(Redo) |
| `IUpdateAnnotationDescriptionUseCase`                    | 주석 설명 수정                  |

모든 데이터 변경(Mutation) 유스케이스는 `Result` 또는 `Result<T>`를 반환하며, `ApplicationError` 코드(`not_found`, `validation`, `persistence`, `unexpected`)를 사용합니다.

## 주요 기능

* 프로젝트 대시보드 및 필터 기능 (전체 / 진행중 / 완료)
* 캡처 작업을 위한 내장 WebView2 브라우저
* 현재 페이지 캡처를 위한 전역 단축키 `Ctrl + Shift + S`
* 폴더 기반 캡처 관리
* 편집기 작업 흐름: 선택 → 작업 영역 정렬 → 번호형 주석 추가
* `%AppData%/Capflow/projects` 경로에 JSON 형태로 데이터 저장

## 요구 사항

* Windows 10 / 11
* .NET 8 SDK
* WebView2 Runtime

## 빌드 및 실행

```powershell
cd d:\intel3\Capflow
dotnet restore Capflow.sln
dotnet build Capflow.sln
dotnet test Capflow.sln
dotnet run --project src\Capflow.Presentation\Capflow.Presentation.csproj
```

## 저장소 구조

```text
%AppData%\Capflow\projects\{project-id}.json      # 메타데이터, 폴더 정보, 주석 데이터
%AppData%\Capflow\assets\{project-id}\images\     # PNG 스크린샷 파일
```
