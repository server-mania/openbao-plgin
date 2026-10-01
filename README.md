# openbao-plugin

OpenBao の外部プラグインを、upstream の公式リリースが出るまでの間だけ
自前ビルドして ghcr.io に配布するためのリポジトリ。

## 背景

OpenBao 2.7.0 で LDAP secrets engine が本体から削除され
([GH-3882](https://github.com/openbao/openbao/pull/3882))、
[openbao-plugins](https://github.com/openbao/openbao-plugins) の外部プラグインに移った。
しかし `secrets/ldap` は main でビルドが通るだけで、リリースタグと OCI イメージがまだ公開されていない
([openbao-plugins#145](https://github.com/openbao/openbao-plugins/issues/145))。

このリポジトリは upstream を **commit SHA で固定**してビルドし、
`ghcr.io/server-mania/openbao-plugin-secrets-ldap` として配布する。
upstream に手を加えたり fork したりはしない。

## 構成

| パス | 内容 |
|---|---|
| `secrets-ldap/upstream.env` | upstream のリポジトリ / commit SHA / 配布バージョン |
| `secrets-ldap/Containerfile` | `FROM scratch` にバイナリ 1 個だけを置くイメージ (upstream と同じ構成) |
| `.github/workflows/secrets-ldap.yaml` | テスト・ビルド・push・provenance 付与 |

## ワークフロー

- **pull_request**: upstream の `make secrets-ldap-test` と `make build` を実行し、
  静的リンクであることを確認してイメージをビルドする (push はしない)。
- **main への push / workflow_dispatch**: 上記に加えて
  1. 既に同じタグがあれば失敗する (タグは上書きしない)
  2. `buildah push` で ghcr.io に push する
  3. `actions/attest-build-provenance` で provenance を付与する
  4. job summary に image digest・バイナリの sha256・OpenBao の plugin stanza を出力する

ビルドするのは `linux_amd64_v1` だけ (k3s ノードは全て amd64)。

## 更新手順

1. `secrets-ldap/upstream.env` の `UPSTREAM_REF` を新しい commit SHA に変える
2. `PLUGIN_VERSION` を `v<CHANGELOG の版>-sm.<SHA 先頭7桁>` に変える
3. PR → main にマージ → job summary から digest と sha256 を取得
4. k3s-apps の OpenBao values.yaml の plugin stanza を更新する

## OpenBao 側の設定例 (2.6.x / 2.7.x 両対応)

```hcl
plugin_directory     = "/openbao/plugins"
plugin_auto_download = true
plugin_auto_register = true   # 2.6.x では既定 false

plugin "secret" "ldap" {
  image       = "ghcr.io/server-mania/openbao-plugin-secrets-ldap"
  version     = "v0.0.1-sm.20f394e"
  binary_name = "openbao-plugin-secrets-ldap"
  sha256sum   = "<job summary の binary sha256>"
}
```

既存の `ldap/` マウントは自動では外部プラグインに切り替わらないので、次を実行する。
`bao plugin reload` は全ノードで実行する必要がある。

```sh
bao secrets tune -plugin-version=v0.0.1-sm.20f394e ldap/
bao plugin reload -mounts=ldap/
```

## 公式リリース後

upstream が `secrets-ldap-vX.Y.Z` をリリースしたら、OpenBao 側の `image` を
`ghcr.io/openbao/openbao-plugin-secrets-ldap` に差し替える。
そのうえでこのリポジトリはアーカイブする。

## ライセンス

ビルド対象の openbao-plugins は MPL-2.0。配布イメージにはそのバイナリだけが含まれる。
