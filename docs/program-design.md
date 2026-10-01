# 作業実績アプリ プログラム構成案

## 1. 設計方針

AndroidアプリはKotlinで実装し、UIにはJetpack Composeを使用する想定です。プログラムはオブジェクト指向を基本とし、画面、業務処理、データ保存、通信を別々のクラスへ分離します。

特に次の方針を守ります。

1. `MainActivity`はアプリの部品を組み立てて最初の画面を呼ぶだけにする。
2. 1つのクラスには、できるだけ1つの責務だけを持たせる。
3. 長い処理を作らず、名前から目的が分かる小さな関数へ分割する。
4. 画面からRoomや通信APIを直接呼ばない。
5. 業務ルールは画面やAndroid固有処理から独立させ、単体テストできるようにする。
6. 処理の理由、注意点、オフライン同期の判断には、人間が読んで理解できる日本語コメントを付ける。
7. コードをそのまま日本語へ置き換えただけのコメントや、実装と重複するコメントは避ける。
8. クラス名、関数名、変数名から役割が分かる英語名を使用する。

## 2. 全体構成

依存関係を一方向に保つため、Presentation、Domain、Dataの3層へ分けます。

```text
┌─────────────────────────────────────┐
│ Presentation                        │
│ Compose画面 / ViewModel / UI状態    │
└──────────────────┬──────────────────┘
                   │ UseCaseを呼ぶ
                   ▼
┌─────────────────────────────────────┐
│ Domain                              │
│ Model / UseCase / Repository IF     │
└──────────────────┬──────────────────┘
                   │ IFを通して利用
                   ▼
┌─────────────────────────────────────┐
│ Data                                │
│ Repository実装 / Room / HTTPS API   │
└─────────────────────────────────────┘
```

- **Presentation**は表示とユーザー操作だけを担当します。
- **Domain**は「作業を開始できるか」「何を送信するか」などの業務ルールを担当します。
- **Data**は端末DBやサーバーAPIとの読み書きを担当します。
- DomainはAndroid、Room、HTTPライブラリのクラスを参照しません。

## 3. ディレクトリ構成

```text
app/src/main/java/<package>/
├── AgriApplication.kt
├── MainActivity.kt
├── AppContainer.kt
│
├── domain/
│   ├── model/
│   │   ├── Field.kt
│   │   ├── WorkRecord.kt
│   │   └── SyncStatus.kt
│   ├── repository/
│   │   ├── FieldRepository.kt
│   │   └── WorkRecordRepository.kt
│   └── usecase/
│       ├── StartWorkUseCase.kt
│       ├── FinishWorkUseCase.kt
│       ├── ObserveActiveWorkUseCase.kt
│       ├── GetRecentWorkContentsUseCase.kt
│       ├── GetWorkHistoryUseCase.kt
│       ├── UpdateWorkTimeUseCase.kt
│       ├── DeleteWorkRecordUseCase.kt
│       └── SendPendingRecordsUseCase.kt
│
├── data/
│   ├── local/
│   │   ├── AgriDatabase.kt
│   │   ├── dao/
│   │   │   ├── FieldDao.kt
│   │   │   └── WorkRecordDao.kt
│   │   ├── entity/
│   │   │   ├── FieldEntity.kt
│   │   │   └── WorkRecordEntity.kt
│   │   └── mapper/
│   │       └── LocalModelMapper.kt
│   ├── remote/
│   │   ├── AgriApi.kt
│   │   ├── dto/
│   │   │   ├── WorkRecordRequest.kt
│   │   │   └── SyncResponse.kt
│   │   └── mapper/
│   │       └── RemoteModelMapper.kt
│   └── repository/
│       ├── DefaultFieldRepository.kt
│       └── DefaultWorkRecordRepository.kt
│
├── presentation/
│   ├── navigation/
│   │   ├── AgriApp.kt
│   │   ├── AppDestination.kt
│   │   └── AppNavHost.kt
│   ├── workentry/
│   │   ├── WorkEntryScreen.kt
│   │   ├── WorkEntryViewModel.kt
│   │   ├── WorkEntryUiState.kt
│   │   └── WorkEntryAction.kt
│   ├── history/
│   │   ├── HistoryScreen.kt
│   │   ├── HistoryViewModel.kt
│   │   └── HistoryUiState.kt
│   ├── edit/
│   │   ├── EditWorkScreen.kt
│   │   ├── EditWorkViewModel.kt
│   │   └── EditWorkUiState.kt
│   └── component/
│       ├── PrimaryActionButton.kt
│       ├── SyncStatusLabel.kt
│       └── WorkRecordCard.kt
│
└── core/
    ├── datetime/
    │   ├── ClockProvider.kt
    │   └── DateTimeFormatter.kt
    ├── id/
    │   └── IdGenerator.kt
    ├── result/
    │   └── AppResult.kt
    └── validation/
        └── WorkRecordValidator.kt
```

テストも実装と対応するディレクトリへ配置します。

```text
app/src/test/java/<package>/
├── domain/usecase/
├── data/repository/
├── presentation/
└── core/validation/
```

## 4. 起動処理

### `MainActivity`の責務

`MainActivity`には業務処理を書きません。Androidの画面を作成し、アプリ全体のCompose関数を呼び出すだけにします。

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        val appContainer = (application as AgriApplication).appContainer

        // Androidの画面領域へ、アプリのルートUIだけを配置する。
        // 作業開始やデータ送信などの処理は各UseCaseが担当する。
        setContent {
            AgriApp(appContainer = appContainer)
        }
    }
}
```

### `AgriApplication`と`AppContainer`

`AgriApplication`はアプリで1つだけ必要な`AppContainer`を生成します。`AppContainer`はDB、API、Repository、UseCaseなどを組み立てる「構成の起点」です。

```kotlin
class AgriApplication : Application() {
    // アプリ全体で共有する依存関係をここから取得する。
    // 各画面がDBやAPIを個別生成しないようにする。
    lateinit var appContainer: AppContainer
        private set

    override fun onCreate() {
        super.onCreate()
        appContainer = DefaultAppContainer(applicationContext)
    }
}
```

MVPでは手動のDependency Injectionでも十分ですが、クラス数が増えた場合はDIライブラリの導入を検討します。どちらの場合も、依存関係の生成場所は`AppContainer`付近へまとめます。

## 5. Domain層

### Domain Model

Domain Modelはアプリ内で扱う意味のあるデータを表します。RoomのEntityやAPIのDTOをそのまま画面へ渡しません。

```kotlin
data class WorkRecord(
    val id: String,
    val field: Field,
    val workContent: String,
    val startedAt: Instant,
    val endedAt: Instant?,
    val syncStatus: SyncStatus,
    val createdAt: Instant,
    val updatedAt: Instant,
    val sentAt: Instant?,
    val deletedAt: Instant?,
) {
    /** 終了時刻がない記録は、端末上で現在進行中の作業として扱う。 */
    val isActive: Boolean
        get() = endedAt == null && deletedAt == null
}
```

### Repository Interface

Domain層には「何ができるか」だけを定義します。RoomやHTTPSの詳細はData層へ隠します。

```kotlin
interface WorkRecordRepository {
    fun observeActiveWork(): Flow<WorkRecord?>
    fun observeHistory(): Flow<List<WorkRecord>>
    fun observePendingCount(): Flow<Int>

    suspend fun hasActiveWork(): Boolean
    suspend fun create(record: WorkRecord)
    suspend fun update(record: WorkRecord)
    suspend fun markDeleted(recordId: String, deletedAt: Instant)
    suspend fun sendPendingRecords(): SyncSummary
}
```

### UseCase

ユーザー操作1つにつき、原則として1つのUseCaseを用意します。UseCaseは小さな関数へ処理を分け、上位関数から読むだけで全体の流れが分かるようにします。

```kotlin
class StartWorkUseCase(
    private val repository: WorkRecordRepository,
    private val validator: WorkRecordValidator,
    private val clock: ClockProvider,
    private val idGenerator: IdGenerator,
) {
    suspend operator fun invoke(field: Field, rawWorkContent: String): AppResult<Unit> {
        val workContent = normalizeWorkContent(rawWorkContent)
        val validation = validator.validateForStart(field, workContent)
        if (validation is ValidationResult.Invalid) {
            return AppResult.Failure(validation.message)
        }

        if (repository.hasActiveWork()) {
            return AppResult.Failure("すでに作業中の記録があります")
        }

        val record = createNewRecord(field, workContent)
        repository.create(record)
        return AppResult.Success(Unit)
    }

    /** 入力時の表記揺れを減らすため、保存前に前後の空白だけを除去する。 */
    private fun normalizeWorkContent(value: String): String = value.trim()

    /** 新しい記録のIDと各日時は、このメソッド内で同じ基準時刻から生成する。 */
    private fun createNewRecord(field: Field, workContent: String): WorkRecord {
        val now = clock.now()
        return WorkRecord(
            id = idGenerator.generate(),
            field = field,
            workContent = workContent,
            startedAt = now,
            endedAt = null,
            syncStatus = SyncStatus.PENDING,
            createdAt = now,
            updatedAt = now,
            sentAt = null,
            deletedAt = null,
        )
    }
}
```

この例のように、公開関数は「正規化→検証→重複確認→生成→保存」という流れだけを示します。詳細は名前の付いた小さな関数や専用クラスへ移します。

## 6. Data層

### Local Data Source

Roomはオフライン動作の正本とします。画面操作は必ず先にRoomへ反映し、送信の成否によって入力データを失わないようにします。

`WorkRecordDao`の主な責務は次のとおりです。

- 作業中の1件を監視する。
- 削除されていない履歴を新しい順で取得する。
- 作業内容を重複除外し、直近5件取得する。
- 未送信件数を監視する。
- 未送信・送信失敗の完了済み記録を取得する。
- 新規、修正、論理削除、送信状態の更新をトランザクションで行う。

### Remote Data Source

`AgriApi`はHTTPS APIの入出力だけを担当します。業務判断や画面用メッセージは持たせません。

```kotlin
interface AgriApi {
    suspend fun sync(request: SyncRequest): SyncResponse
}
```

APIへ送るDTOとRoom Entityは分離し、Mapperクラスで相互変換します。これにより、MySQL側のAPI形式を変更しても画面やDomain Modelへの影響を限定できます。

### Repository Implementation

`DefaultWorkRecordRepository`がRoomとAPIを調整します。ただし大きなクラスにしないため、取得、ローカル更新、送信結果反映を小さな非公開関数へ分割します。

```kotlin
override suspend fun sendPendingRecords(): SyncSummary {
    val targets = loadSyncTargets()
    if (targets.isEmpty()) return SyncSummary.empty()

    markAsSending(targets)
    return sendAndApplyResult(targets)
}
```

同期処理では次を守ります。

1. 作業中の記録は送らない。
2. 送信対象を`SENDING`へ変更してからAPIを呼ぶ。
3. APIが成功を返した記録だけを`SENT`へ変更する。
4. 失敗した記録は`FAILED`へ変更し、次回送信対象に残す。
5. 送信中に記録が編集された場合、古い送信結果で新しい内容を`SENT`にしない。
6. UUIDと`updated_at`を使用して、再送と古い更新の上書きを防ぐ。

## 7. Presentation層

### 単方向データフロー

画面は`UiState`を表示し、操作を`Action`としてViewModelへ渡します。

```text
ユーザー操作
    │
    ▼
WorkEntryAction
    │
    ▼
WorkEntryViewModel ──→ UseCase
    │                     │
    │                     ▼
    │                 Repository
    ▼
WorkEntryUiState
    │
    ▼
WorkEntryScreen
```

### UI State

画面に必要な値を1つの不変オブジェクトへまとめます。

```kotlin
data class WorkEntryUiState(
    val fields: List<Field> = emptyList(),
    val selectedField: Field? = null,
    val workContent: String = "",
    val recentWorkContents: List<String> = emptyList(),
    val activeWork: WorkRecord? = null,
    val pendingCount: Int = 0,
    val isSaving: Boolean = false,
    val isSending: Boolean = false,
    val message: UiMessage? = null,
) {
    val canStart: Boolean
        get() = activeWork == null &&
            selectedField != null &&
            workContent.isNotBlank() &&
            !isSaving

    val canSend: Boolean
        get() = pendingCount > 0 && !isSaving && !isSending
}
```

### Action

画面からViewModelへ渡す操作を列挙し、クリック処理を画面内へ書き散らさないようにします。

```kotlin
sealed interface WorkEntryAction {
    data class SelectField(val field: Field) : WorkEntryAction
    data class ChangeWorkContent(val value: String) : WorkEntryAction
    data class SelectRecentWork(val value: String) : WorkEntryAction
    data object StartWork : WorkEntryAction
    data object ConfirmFinishWork : WorkEntryAction
    data object SendPendingRecords : WorkEntryAction
    data object DismissMessage : WorkEntryAction
}
```

### ViewModel

ViewModelはActionを受け取り、対応する小さな処理へ振り分けます。DBのSQLやHTTP通信処理は書きません。

```kotlin
fun onAction(action: WorkEntryAction) {
    when (action) {
        is WorkEntryAction.SelectField -> selectField(action.field)
        is WorkEntryAction.ChangeWorkContent -> changeWorkContent(action.value)
        is WorkEntryAction.SelectRecentWork -> selectRecentWork(action.value)
        WorkEntryAction.StartWork -> startWork()
        WorkEntryAction.ConfirmFinishWork -> finishWork()
        WorkEntryAction.SendPendingRecords -> sendPendingRecords()
        WorkEntryAction.DismissMessage -> dismissMessage()
    }
}
```

### Compose Screen

画面関数は状態を受け取り、表示とイベント通知だけを行います。

```kotlin
@Composable
fun WorkEntryScreen(
    state: WorkEntryUiState,
    onAction: (WorkEntryAction) -> Unit,
    modifier: Modifier = Modifier,
) {
    WorkEntryContent(
        state = state,
        onFieldClick = { /* 圃場選択ダイアログを表示する */ },
        onWorkContentChange = {
            onAction(WorkEntryAction.ChangeWorkContent(it))
        },
        onStartClick = {
            onAction(WorkEntryAction.StartWork)
        },
        onSendClick = {
            onAction(WorkEntryAction.SendPendingRecords)
        },
        modifier = modifier,
    )
}
```

大きな画面関数は、`FieldSelector`、`RecentWorkChips`、`ActiveWorkCard`、`SyncSection`のような意味のある部品へ分割します。単に行数を減らすためだけの関数分割はせず、部品名が画面の役割を表す単位にします。

## 8. クラス別責務

| クラス | 責務 | 持たせない処理 |
| --- | --- | --- |
| `MainActivity` | Composeアプリの起動 | DB、通信、業務判断 |
| `AppContainer` | 依存オブジェクトの生成・接続 | 画面状態、業務処理 |
| `WorkEntryScreen` | 作業入力状態の表示と操作通知 | DB、通信、入力保存 |
| `WorkEntryViewModel` | UI状態管理とUseCase呼び出し | SQL、API実装 |
| `StartWorkUseCase` | 開始条件の確認と記録作成 | 画面表示、SQL |
| `FinishWorkUseCase` | 作業中記録の終了 | ダイアログ表示 |
| `UpdateWorkTimeUseCase` | 日時の検証と修正 | 日時選択UI |
| `DeleteWorkRecordUseCase` | 送信状態に応じた削除判断 | 削除確認UI |
| `SendPendingRecordsUseCase` | 手動同期の開始と結果整理 | HTTPの詳細 |
| `DefaultWorkRecordRepository` | RoomとAPI間のデータ調整 | Compose状態 |
| `WorkRecordDao` | SQLiteへの問い合わせ | 業務メッセージ |
| `AgriApi` | HTTPS APIとの通信 | DB保存、UI判断 |

## 9. 関数分割の基準

次のいずれかに該当したら、別関数または別クラスへ分けます。

- 関数名を「AしてBする」のように2つ以上の動詞で説明する必要がある。
- 入力チェック、データ変換、保存、通信が1つの関数に混在している。
- 同じ処理が2か所以上に現れる。
- `if`や`when`の入れ子が深くなり、正常な流れが読み取りにくい。
- テストで確認したい規則がprivateな巨大処理に埋もれている。
- コメントがないと処理のまとまりを説明できない。

目安として、1関数は1つの目的に絞ります。ただし「必ず何行以下」のような機械的な分割は行わず、呼び出し側を読んだときに処理の流れが自然に分かることを優先します。

## 10. コメント規約

### コメントを書く場所

- オフライン同期や論理削除など、採用理由がコードだけでは分からない箇所。
- 再送、競合、日時処理など、将来変更すると不具合につながりやすい箇所。
- publicクラス・関数の責務、引数、戻り値、失敗条件。
- 一見不要に見える判定を残している理由。
- さくらのAPIやMySQLとの契約上、守る必要がある仕様。

### 避けるコメント

```kotlin
// 未送信件数を1増やす
pendingCount += 1
```

上のようにコードを読み上げるだけのコメントは追加しません。代わりに、判断理由を書きます。

```kotlin
// 送信中に編集された内容を、古いAPI応答で送信済みにしない。
// 送信開始時と現在のupdatedAtが一致する場合だけ状態を更新する。
if (current.updatedAt == sentVersion.updatedAt) {
    markAsSent(current.id)
}
```

コメントは日本語を基本とし、処理変更時にはコードと同時に更新します。

## 11. エラー処理

- 例外を画面までそのまま渡さず、`AppResult`や用途別のエラーへ変換します。
- DB保存失敗、入力不正、オフライン、HTTPエラー、APIの一部失敗を区別します。
- ユーザー向けメッセージと、開発者向けログを分けます。
- APIキー、認証ヘッダー、作業内容などの個人データをログへ出しません。
- `try/catch`は例外を意味のある結果へ変換できる境界に置き、各関数へ重複して書きません。

## 12. テスト構成

### Unit Test

- `StartWorkUseCaseTest`: 空文字、作業中、正常開始。
- `FinishWorkUseCaseTest`: 作業なし、正常終了、保存失敗。
- `GetRecentWorkContentsUseCaseTest`: 新しい順、重複除外、最大5件、削除済み除外。
- `UpdateWorkTimeUseCaseTest`: 終了が開始以前、未来日時、正常修正。
- `DeleteWorkRecordUseCaseTest`: 未送信の物理削除、送信済みの論理削除。
- `SendPendingRecordsUseCaseTest`: 全件成功、一部失敗、再送、送信中編集。
- `WorkEntryViewModelTest`: ActionごとのUiState変化。

### Database Test

- 作業中レコードを同時に2件作成できない。
- 履歴が新しい順に取得できる。
- 作業候補が重複なしで最大5件取得できる。
- 削除済みレコードが通常の履歴に表示されない。

### UI Test

- 入力不足時に開始ボタンが無効になる。
- 開始後に作業中表示へ切り替わる。
- 終了・削除で確認ダイアログが表示される。
- 履歴から編集画面へ移動できる。
- 未送信件数と送信結果が表示される。

## 13. 実装順

1. GradleとAndroidプロジェクトの基本構成を作成する。
2. Domain Model、Repository Interface、日時・IDの抽象化を作る。
3. Room Entity、DAO、Database、Mapperを作る。
4. UseCaseとUnit Testを作る。
5. ViewModel、UiState、Actionを作る。
6. Composeの共通部品と各画面を作る。
7. 画面とローカルDBを接続し、完全オフラインで動作確認する。
8. HTTPS API、DTO、Repositoryの同期処理を作る。
9. さくらのレンタルサーバーとMySQLへ接続する。
10. Unit Test、Database Test、UI Test、実機テストを通す。

## 14. コーディング時の確認項目

- `MainActivity`に業務処理が入っていないか。
- Compose画面からDAOやAPIを直接呼んでいないか。
- ViewModelにSQLやHTTPの詳細が入っていないか。
- クラスと関数の名前だけで目的が推測できるか。
- 1つの関数へ検証、保存、通信、表示更新を詰め込んでいないか。
- 複雑な判断に「何をしているか」ではなく「なぜ必要か」のコメントがあるか。
- 送信失敗後も端末の作業記録が残るか。
- 送信中の編集や再送で、新しい内容が古い内容へ戻らないか。
- テストしにくい日時、UUID、通信処理が抽象化されているか。
