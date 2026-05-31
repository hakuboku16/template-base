# Python テンプレートプロジェクト 設計書

作成日: 2026-05-20

## 目的

- 複数プロジェクトで大まかなフォルダ構成を統一する
- 新規プロジェクト着手時に、ログ出力・設定値読み込みなどの共通基盤を即利用できる状態にする
- 着手から「実装したい機能のコード」までの距離を最短化する

## スコープ

含む:
- 共通基盤(`src/core/logger.py`, `src/core/config.py`)
- 環境別(development / test / production)の設定切替機構
- `pyproject.toml`(uv 管理)
- Docker 実行環境(`Dockerfile`, `docker-compose.yml`)
- `.env.example` と `.gitignore`
- `pytest` の最小雛形

含まない:
- 認証・DB 接続・Web フレームワーク等のドメイン機能
- CI/CD パイプライン定義
- Linter/Formatter 設定(ruff 等。必要になった時点で追加)

## 全体ディレクトリ構成

```
template/
├── .env.example              # 設定値のサンプル
├── .gitignore
├── .dockerignore
├── Dockerfile
├── docker-compose.yml
├── pyproject.toml            # uv 管理
├── README.md
├── docs/
├── logs/
│   └── .gitkeep
├── src/
│   ├── __init__.py
│   ├── main.py               # エントリーポイント
│   └── core/
│       ├── __init__.py
│       ├── config.py         # pydantic-settings(環境別 Settings)
│       └── logger.py         # setup_logger()
└── tests/
    ├── __init__.py
    ├── conftest.py
    └── test_config.py
```

採用しなかった案:
- 機能別フォルダ(services/models/utils)を先に切る案 → 空フォルダが残り、テンプレ目的に対し過剰
- フラット構成(src/ なし) → import 経路が短いがパッケージ化に弱く、テストで sys.path 操作が必要になりがち

## 設定値の読み込み(config.py)

### 方針

- `.env` はシークレットと `APP_ENV` のみを管理する。`log_level` 等の非機密設定は `config.py` のクラスデフォルトで管理する
  - 環境ごとの差分は `DevSettings` / `TestSettings` / `ProdSettings`(`BaseAppSettings` を継承)で表現する
  - 一時的に上書きしたい場合は OS 環境変数で渡す(例: `LOG_LEVEL=DEBUG python -m src.main`)。pydantic-settings は OS 環境変数も自動で読む
- `get_settings()` で `APP_ENV` を見て該当クラスのインスタンスを返す
- `@functools.lru_cache` でシングルトン化

### クラス構造

```python
class AppEnv(str, Enum):
    DEVELOPMENT = "development"
    TEST = "test"
    PRODUCTION = "production"

class BaseAppSettings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        extra="ignore",
    )
    app_env: AppEnv = AppEnv.DEVELOPMENT
    log_level: str = "INFO"
    log_dir: str = "logs"
    log_file_name: str = "python.log"
    log_backup_days: int = 30

class DevSettings(BaseAppSettings):
    log_level: str = "DEBUG"

class TestSettings(BaseAppSettings):
    log_level: str = "DEBUG"
    log_file_name: str = "test.log"

class ProdSettings(BaseAppSettings):
    log_level: str = "INFO"

@lru_cache
def get_settings() -> BaseAppSettings:
    env = BaseAppSettings().app_env
    mapping = {
        AppEnv.DEVELOPMENT: DevSettings,
        AppEnv.TEST: TestSettings,
        AppEnv.PRODUCTION: ProdSettings,
    }
    return mapping[env]()

settings = get_settings()
```

### 使用例

```python
from src.core.config import settings
print(settings.log_level)
```

## ロガー(logger.py)

### 方針

- `setup_logger()` をプロセス起動時(`main.py`)に **1 回だけ** 呼ぶ
- ルートロガーにコンソール・ファイル両方のハンドラを設定
- 以降は各モジュールで `logging.getLogger(__name__)` を使うだけで、上記設定が継承される
- ファイル出力は `TimedRotatingFileHandler`(日次ローテーション、毎日深夜に切替)。保存日数は `config` から読む
- 多重初期化を防ぐため、既存ハンドラは `setup_logger()` 内でクリア

### 関数構造

```python
def setup_logger() -> None:
    log_dir = Path(settings.log_dir)
    log_dir.mkdir(parents=True, exist_ok=True)
    log_path = log_dir / settings.log_file_name

    root = logging.getLogger()
    root.setLevel(settings.log_level)
    root.handlers.clear()

    formatter = logging.Formatter(
        "%(asctime)s [%(levelname)s] %(name)s - %(message)s"
    )

    console = logging.StreamHandler()
    console.setFormatter(formatter)
    root.addHandler(console)

    file_handler = TimedRotatingFileHandler(
        log_path,
        when="midnight",
        interval=1,
        backupCount=settings.log_backup_days,
        encoding="utf-8",
    )
    file_handler.setFormatter(formatter)
    root.addHandler(file_handler)
```

### 使用例

`main.py`:
```python
import logging
from src.core.logger import setup_logger

setup_logger()
logger = logging.getLogger(__name__)
logger.info("application started")
```

他モジュール:
```python
import logging
logger = logging.getLogger(__name__)
logger.debug("...")
```

## 環境切り替えの動作

`APP_ENV` の値で `get_settings()` がインスタンス化するクラスを決定する。

| 設定方法          | 例                                                             |
| ----------------- | -------------------------------------------------------------- |
| `.env` ファイル   | `APP_ENV=production` を記載                                    |
| OS 環境変数 (CLI) | `APP_ENV=test python -m src.main`                              |
| Docker            | `docker-compose.yml` の `environment:` で指定                  |
| pytest            | `tests/conftest.py` で `os.environ["APP_ENV"] = "test"` を強制 |

未指定時は `development`。

## パッケージ管理(uv)

`pyproject.toml` に最小依存を記載:
- `pydantic-settings`
- 開発依存として `pytest`

`uv.lock` は初回 `uv sync` 実行時に自動生成。

## Docker 構成

- `Dockerfile`: 公式 Python ベースイメージ + uv で依存導入 + `src/main.py` を実行
- `docker-compose.yml`: サービス 1 つ(`app`)、`environment: APP_ENV=development` を例示
- `.dockerignore`: `.env`, `logs/`, `__pycache__/`, `.git/` 等を除外

## .env.example の内容

`.env` にはシークレットと `APP_ENV` のみを置く。非機密設定(`log_level` 等)は `config.py` のクラスデフォルトで管理し、ここには記載しない。

```
# 環境切替(必須)
APP_ENV=development

# シークレット類はここに追記(例)
# DATABASE_URL=
# API_KEY=
```

## .gitignore の主な内容

```
.env
__pycache__/
*.py[cod]
logs/*.log
logs/*.log.*
.venv/
.pytest_cache/
.mypy_cache/
```

## テスト戦略

- `tests/conftest.py` で `APP_ENV=test` を強制(本番設定を誤って読まない安全装置)
- `tests/test_config.py`:
  - `APP_ENV=test` のときに `TestSettings` が返ることを検証
  - 他環境を `monkeypatch` で切り替えた際の挙動を検証
- ログ周りの単体テストは設計上含めない(ハンドラ設定の検証は副作用が大きいため、必要時に追加)

## 受け入れ条件

1. `uv sync` で依存導入が完了する
2. `python -m src.main`(または `docker compose up`)が起動し、コンソール・`logs/python.log` の両方にログが出る
3. `APP_ENV` を変えると `settings.log_level` 等の値が変わることが確認できる
4. `pytest` が緑(サンプルテストが通る)
5. 新規プロジェクト時は `template2` をコピー → `.env.example` を `.env` にリネーム → すぐ実装に入れる
