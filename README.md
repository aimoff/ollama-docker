# Dockerfile for Ollama on Linux with AMD GPU
[Ollama][] をローカル PC で実行するための Docker 環境構築ファイル群

[Ollama]: https://ollama.com/

手元の Linux ホスト環境をあまりいじらずに [Ollama][] を試せるようにしたもの

実はこれを作成した後で [オフィシャルのコンテナイメージ][Dockerhub ollama] があることに気づいたが、
自分の使い方と合わないところがあるのでそのまま晒すことにする

[Dockerhub ollama]: https://hub.docker.com/r/ollama/ollama

## オフィシャルの ollama コンテナイメージとの違い
- 扱いやすいように docker-compose.yml を用意
- [Open WebUI][] も起動し WebUI 経由でも Ollama を使用できるようにしている
- ollama service を root ではなく ollama ユーザで実行
   - これを実現するために [Supervisord][] を使用 ([Use a process manager][])
- exec でコンテナ内コマンドを実行する際にもユーザ権限で実行するようにする
- オフィシャルの ubuntu ベースだけでなく debian ベースも用意  
  …というよりも debian 版を先に作って後から ubuntu 版も同様に動くようにした
   - ubuntu 版: オフィシャルの ollama コンテナをベースに上記変更だけ反映
      - 自分の環境が AMD GPU のためベースイメージは ollama/ollama:rocm にしている
   - debian 版: Debian Trixie のコンテナをベースをベースに
     [Ollama の Linux 版][Download Ollama] をインストールして設定

[Supervisord]: https://supervisord.org/
[Use a process manager]: https://docs.docker.com/engine/containers/multi-service_container/#use-a-process-manager
[Download Ollama]: https://ollama.com/download/linux
[Open WebUI]: https://docs.openwebui.com/

## Ollama コンテナイメージの Build
### `.env` の修正
使用している Linux ホスト環境に合わせて `.env` を修正する

- `OS` : 作成するコンテナイメージのベース OS (`ubuntu` or `debian`)
- `TAG` : コンテナイメージに付与するラベル
- `UID` : Ollama をインタラクティブに実行する際のユーザ ID
   - 使用者の Linux ホストの UID を指定するとパーミッション的に扱いやすい
   - 自身の UID は `id -u` コマンドで取得できる
- `RENDER_GID` 【必須】: Linux ホストの `render` グループ ID
   - 値は `getent group render` コマンドで確認できる
   - この値が間違っていると GPU が使えなくなる
- `USER_NAME` : Ollama をインタラクティブに実行する際のユーザ名
   - `UID` に対応したユーザ名
   - これを指定しないと `user` というユーザが作られる
   - ubuntu 版で UID = 1000 の場合は既存の `ubuntu` ユーザがそのまま使われる（名前だけ変更）
   - ユーザ名の指定そのものにはさしたる意味はないが、
     実行時にホームディレクトリをマウントする際に使われる  
     使用者の Linux ホストのユーザ名と合わせると Drag & Drop の際に都合がいい
- `LANG` : 実行環境の locale
   - これを指定しなければ Linux ホストの LANG 環境変数が渡される
   - 日本語ファイル名をリストする際等で文字化けしないようにするため

### Build
`docker compose` (或いは `docker-compose`) コマンドを使って Build する

```console
$ docker compose build
```

## 設定
### `docker-compose.yml` の修正
必要に応じて `docker-compose.yml` ファイルを修正する

#### `ollama` service の `environment` セクション
AMD の GPU で公式にサポートされているのは [Supported GPUs][] だけ  
しかしながら実際には [Compatibility matrix][] にあるように
RDNA2 〜 RDNA4 アーキであれば動いてくれる可能性がある

Linux ホストに `rocminfo` パッケージがインストールされていれば以下の手順で GPU のアーキがわかる

```console
$ rocminfo | grep gfx
  Name:                    gfx1031                            
      Name:                    amdgcn-amd-amdhsa--gfx1031       
```

表示された `gfxXXXX` の LLVM target が [Supported GPUs][] にリストされていない場合には
近い LLVM target に偽装するために以下の環境変数を指定する

- `GFX_ARCH` : 偽装する LLVM target
   - 例: `gfx1030`
- `HSA_OVERRIDE_GFX_VERSION` : LLVM target の数字部分をバージョンに読み替えたもの
   - 例: `10.3.0`

※ 偽装しても動かない可能性はある

AMD GPU アーキの詳細については [User Guide for AMDGPU Backend][] を参照

[Supported GPUs]: https://rocm.docs.amd.com/projects/install-on-linux/en/docs-6.4.3/reference/system-requirements.html#supported-gpus
[Compatibility matrix]: https://rocm.docs.amd.com/en/docs-6.4.3/compatibility/compatibility-matrix.html
[User Guide for AMDGPU Backend]: https://llvm.org/docs/AMDGPUUsage.html#processors

参考: [Pytorch Performance on AMD Radeon and Instinct GPUs](https://www.amd.com/content/dam/amd/en/documents/developer/webinars/rocm/pytorch-on-radeon-and-instinct-gpus.pdf) Page 20

#### `ollama` service の `volumes` セクション
一般的な [volumes][] の記述方法に従い mount する volume を指定する  
このコンテナイメージ固有の mount target は以下

- `/home/ollama` : バックグラウンドの ollma service が使用するディレクトリ
   - `ollama run` や `ollama pull` によりダウンロードされるモデルの格納先
   - モデルは大容量となることがあるため、空き容量の多いストレージを指定する
- `/home/${USER_NAME}` : コマンドライン実行時のワークディレクトリ
   - ホストと Docker 内とのファイル受け渡しに使う
   - 使用者の Linux ホストのホームディレクトリを mount 元にすると
     CLI のターミナルに Drag & Drop した時に矛盾しなくなる
   - セキュリティ的には専用のディレクトリを用意した方が安全

[volumes]: https://docs.docker.com/reference/compose-file/services/#volumes

#### `webui` service の `environment` セクション
必要に応じて [環境変数][Environment Variable Configuration] を設定する

- [`OLLAMA_BASE_URL`][OLLAMA_BASE_URL] : Ollama API へのアクセス先 URL
   - 変更不要
   - ネット上では古い `OLLAMA_API_BASE_URL` の方を書いているものが多いので要注意
- [`WEBUI_SECRET_KEY`][WEBUI_SECRET_KEY] : Web Token に使用する Secret key
   - デフォルトのままだと何も指定していないので、サービス起動の度に毎回新しい key がランダムに生成される
   - 個人で使用している場合には問題にならないかもしれないが、設定しておいたほうがベター
   - 値は `docker-compose.yml` に直接書かずに `.env` に書く
- [`WEBUI_AUTH`][WEBUI_AUTH] : 初回 WebUI アクセス時に管理者アカウントを登録させるか否か
   - お試しではこの環境変数に *False* を設定するのが楽
   - 認証なしにアクセスできるので当然セキュリティ的には良くない
   - ひとまず *False* に設定した上で管理者パネルからユーザを追加した後に *True* に戻すのが望ましい

[Environment Variable Configuration]: https://docs.openwebui.com/getting-started/env-configuration
[OLLAMA_BASE_URL]: https://docs.openwebui.com/getting-started/env-configuration#ollama_base_url
[WEBUI_SECRET_KEY]: https://docs.openwebui.com/getting-started/env-configuration#webui_secret_key
[WEBUI_AUTH]: https://docs.openwebui.com/getting-started/env-configuration#webui_auth

#### `webui` service の `ports` セクション
必要に応じポート番号の `3000` を変更する

この設定では localhost からしか WebUI にアクセスできない  
他のクライアントからもアクセスしたい場合でも、
この ports の IP アドレス設定は変えずにホスト上に
nginx 等でリバースプロキシを構築するのが望ましい  
※ Docker はファイアウォールをバイパスして port を公開してしまうため

#### `webui` service の `volumes` セクション
一般的な [volumes][] の記述方法に従い Open WebUI の使用する volume を指定する

Ollama ほどの容量は必要ないかもしれないが GB オーダーの空き領域があるストレージを指定する

### `.env` の修正
必要に応じ `.env` を修正する

- `WEBUI_TAG` : Open WebUI のコンテナイメージを指定する Tag
   - 指定しなければ `main` Tag のイメージが選択される
   - `slim` は若干サイズが小さくなるが大差ない
- `WEBUI_SECRET_KEY` : enviroment セクションに反映させるための Secret key 指定
   - `openssl rand -base64 32` コマンド等で作成したエントロピーの高い文字列を指定する

## 実行
### サービスの起動
まず ollama と webui サービスをバックグラウンドで動かす

```console
$ docker compose up -d
```

webui が使えるようになるまでに数分かかることがあるので、
起動後すぐにアクセスせずにしばらく待つ  
`docker compose ps` で webui が healthy となればアクセス可能

```console
$ docker compose ps
```

### CLI
Ollama を [CLI][] で操作する

[CLI]: https://docs.ollama.com/cli

#### Docker 内のシェルを起動
```console
$ docker compose exec ollama bash
```

Docker 内のシェルで [Ollama CLI][CLI] だけでなく各種コマンドを実行できる

```console
$ ls
$ ollama ls
$ ollama run gemma3
```

#### 直接 Ollama のモデルを実行
以下は gemma3 の例

```console
$ docker compose exec ollama ollama run gemma3
```

### WebUI
サービスを起動したホスト上で Web ブラウザから WebUI にアクセスする

URL: http://localhost:3000 (ポート番号を変更している場合にはその値)

詳細は [Open WebUI][] のドキュメント等を参照

モデルは WebUI からは登録できないので [CLI][] から登録（ローカルにダウンロード）する

```console
$ docker compose exec ollama ollama pull gpt-oss-20b
```
