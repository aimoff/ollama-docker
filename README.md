# Dockerfile for Ollama on Linux with AMD GPU
[Ollama][] をローカル PC で実行するための Docker 環境構築ファイル群

[Ollama]: https://ollama.com/

手元の Linux ホスト環境をあまりいじらずに [Ollama][] を試せるようにしたもの

実はこれを作成した後で [オフィシャルの Docker image][Dockerhub ollama] があることに気づいたが、
自分の使い方と合わないところがあるのでそのまま晒すことにする

[Dockerhub ollama]: https://hub.docker.com/r/ollama/ollama

## [オフィシャルの Docker image][Dockerhub ollama] との違い
- 扱いやすいように docker-compose.yml を用意
- ollama service を root ではなく ollama ユーザで実行
   - これを実現するために [Supervisord][] を使用 ([Use a process manager][])
- exec でコンテナ内コマンドを実行する際にもユーザ権限で実行するようにする
- オフィシャルの ubuntu ベースだけでなく debian ベースも用意  
  …というよりも debian 版を先に作って後から ubuntu 版も同様に動くようにした
   - ubuntu 版: オフィシャルの Docker image をベースに上記変更だけ反映
      - 自分の環境が AMD GPU のためベースイメージは ollama/ollama:rocm にしている
   - debian 版: Debian Trixie の Docker image をベースに
     [Ollama の Linux 版][Download Ollama] をインストールして設定

[Supervisord]: https://supervisord.org/
[Use a process manager]: https://docs.docker.com/engine/containers/multi-service_container/#use-a-process-manager
[Download Ollama]: https://ollama.com/download/linux

## Build 方法
### 1. `.env` を修正
使用している Linux ホスト環境に合わせて `.env` を修正する

- `OS` : 作成する Docker image の OS (`ubuntu` or `debian`)
- `TAG` : Docker image に付与するラベル
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

### 2. Build
`docker compose` (或いは `docker-compose`) コマンドを使って Build する

```console
$ docker compose build
```

## 実行
### `docker-compose.yml` の修正
必要に応じて `docker-compose.yml` ファイルを修正する

#### `environment` セクション
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

[Supported GPUs]: https://rocm.docs.amd.com/projects/install-on-linux/en/docs-6.4.3/reference/system-requirements.html#supported-gpus
[Compatibility matrix]: https://rocm.docs.amd.com/en/docs-6.4.3/compatibility/compatibility-matrix.html

参考: [Pytorch Performance on AMD Radeon and Instinct GPUs](https://www.amd.com/content/dam/amd/en/documents/developer/webinars/rocm/pytorch-on-radeon-and-instinct-gpus.pdf) Page 20

#### `volumes` セクション
一般的な [volumes][] の記述方法に従い mount する volume を指定する  
この Docker image 固有の mount target は以下

- `/home/ollama` : バックグラウンドの ollma service が使用するディレクトリ
   - `ollama run` や `ollama pull` によりダウンロードされるモデルの格納先
   - モデルは大容量となることがあるため、空き容量の多いストレージを指定する
- `/home/${USER_NAME}` : コマンドライン実行時のワークディレクトリ
   - ホストと Docker 内とのファイル受け渡しに使う
   - 使用者の Linux ホストのホームディレクトリを mount 元にすると Drag & Drop で矛盾しなくなる
   - セキュリティ的には専用のディレクトリを用意した方が安全

[volumes]: https://docs.docker.com/reference/compose-file/services/#volumes

### ollama サービスの起動
まず ollama サービスをバックグラウンドで動かす

```console
$ docker compose up -d
```

### [CLI][] の実行
Docker 内で実行したいコマンドを指定する

[CLI]: https://docs.ollama.com/cli

#### Docker 内でシェルを起動
```console
$ docker compose exec ollama bash
```

Docker 内のシェルで各種コマンドを実行できる

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
