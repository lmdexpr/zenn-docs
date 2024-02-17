---
title: "Auth0+Discord で MinIO へ SSO する"
emoji: "🦩"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: 
- "auth0"
- "Discord"
- "MinIO"
published: true
---
# 背景
運営している Discord サーバーにて、録音をとることがあるのですがファイルサーバーを用意したので良い感じにしたいな～と思っていました。
API とかで操作しやすいと嬉しいので [MinIO](https://min.io) が良さそうということで採用しました。

ところで、権限についても Discord に結びついているととても嬉しいです。
特に複数の Discord サーバーがあり、それぞれで権限設定を変えたい、という背景もありました。

# やりたいこと
Discord サーバー A, B があります。
A, B 両方に所属している人もいれば、片方だけの人もいます。

一方で MinIO にはそれぞれ対応したバケット bucket_A, bucket_B があります。

A 所属の人は bucket_A のみが、B 所属の人は bucket_B のみが、両方所属の人は両方のバケットが見えてほしいです。

# やったこと

## Discord による認可について (なぜ Auth0 も使うのか)
Discord が OIDC IdP として利用できれば良いのですが、提供されているのは [OAuth2.0 だけ](https://discord.com/developers/docs/topics/oauth2)です。
OAuth2.0 と OIDC の違いはここで説明しませんが、OAuth2.0 は認証に使えないため、Discord だけでサインインさせることは出来ません。
https://www.sakimura.org/2012/02/1487/

MinIO が対応しているのも OIDC です。
そのため、Auth0 を認証のための OIDC IdP として利用した上で認可に Discord を使用します。

## Discord 側の設定をする
[Discord Developer Portal](https://discord.com/developers/applications) で Application を作成します。

OAuth2 タブの中にある redirects を設定しておきます。
`https://<なんとかかんとか>.jp.auth0.com/login/callback` みたいな形式です。

同じタブの中にある ClientID/ClientSecret を確認しておきます。
後で必要になります。

## Auth0 側の設定をする
幸いなことに Auth0 と Discord の連携は用意された integration があります。
integration: https://marketplace.auth0.com/integrations/discord-social-connection

サーバーの情報が取得できないなど、痒い所に手が届かない部分もあります。
が、今回は別の方法でサーバーに関する設定を行うので一旦この integration を使用します。

integration には ClientID/ClientSecret が必要なので先ほど確認した内容を入力します。

設定が出来たら Auth0 側で Applications を作成し、作成した discord integration のみを使用するようにします。
この Applications の ClientID/ClientSecret も後で必要になります

## MinIO 側の設定をする
Identity>OpenID という項目があるので作成します。
以下の設定にします。

- Config URL
  - `https://<なんとかかんとか>.jp.auth0.com/.well-known/openid-configuration`
- Client ID
  - Auth0 のやつ
- Client Secret
  - Auth0 のやつ
- Claim Name
  - `https://<mydomain>/policy`
- Scopes
  - `openid, identify, https://<mydomain>/policy`
  - (正直よく分かってないです。要らない気もする)
- Redirect URI
  - `https://<mydomain>/oauth_callback`

ここで重要なのは Claim Name です。
MinIO の OpenID 設定のこの項目はどうやらこの認証で使用される policy を指定できるようです。
ドキュメント上からちゃんと仕様を確認できていないのですが、参考資料にそれっぽいことが書いてあります。
実際、使用できました。
変な指定になっているのはこの後説明します。

https://min.io/docs/minio/macos/operations/external-iam/configure-keycloak-identity-management.html

また、ポリシーを作っておく必要があるので、bucket_A、 bucket_B のためのポリシー policy_A、policy_B を適当に作っておきます。
この辺りの詳細は文書の範囲外とします。（面倒なので）

## サーバーに所属しているか確認する
ここまでで MinIO から実際に Auth0, Discord を経由し、認証を試みることが出来ます。
しかし、指定した claim に値がないのでポリシーを取得できず、エラーが発生します。

ここで Auth0 の [Login Flow](https://auth0.com/docs/customize/actions/flows-and-triggers/login-flow) を使います。
これはログイン時などにちょっとしたコードを動かせる機能です。便利。

ここで Discord のサーバーに所属しているかや、ポリシーの設定を行います。
具体的には以下のコードでやりました。

また、コードの前半は以下の記事を参考にしております。
大変助かりました。
https://musaprg.hatenablog.com/entry/2023/02/23/180455

コードの後半は `discord.js@latest` を使ったせいで自分で書くことになりました。悲しい。

```js
const { Client, GatewayIntentBits } = require('discord.js');

exports.onExecutePostLogin = async (event, api) => {
  const identities = event.user.identities.filter((v) => { return v.connection == "discord"; });
  const identity = identities.length == 1 ? identities[0] : null;
  if (!identity) {
    api.access.deny("this user isn't authenticated with discord");
    return;
  }
  const result = identity.user_id?.match(/discord\|(.+)/);
  if (result?.length != 2) {
    api.access.deny("the format of user_id is invalid");
  }
  const user_id = result?.length == 2 ? result[1] : null;

  const client = new Client({
	  intents: [
		  GatewayIntentBits.Guilds,
		  GatewayIntentBits.GuildMembers,
    ],
  });
  client.login(event.secrets.DISCORD_BOT_TOKEN);
  async function joined(server_id) {
    const server = await client.guilds.fetch(server_id);
    return await server.members.fetch(user_id)
      .then(() => { return server_id; })
      .catch((e) => { console.log(e); return null; });;
  }
  const policy = [
    await joined('<A server id>'),
    await joined('<B server id>')
  ].filter(v => v);

  if (!policy.length) {
    api.access.deny("You are not a member. Please contact the administrator for details.");
    return;
  }
  const namespace = 'https://<mydomain>/';
  api.idToken.setCustomClaim(namespace +"policy", policy);
};
```

要所だけ説明します。
まず、ログイン後に discord を使用しているかなどを確認し、API を用いてサーバーの情報を取得します。
ここで、Discord Bot のトークンが必要になったり、その Bot がサーバー A, B に所属している必要があります。

その後、所属していなければ deny します。
所属している場合には `setCustomClaim` で claim にポリシーを入れます。
ここで、ネームスペースを区切るために自分のドメインを入れろ、とドキュメントにあります。
大人しく従っておきましょう。

https://auth0.com/docs/get-started/apis/scopes/sample-use-cases-scopes-and-claims

自分は今後の拡張を考えてサーバー ID をそのままポリシーの名前にしました。
クレデンシャル情報かどうか微妙なところなので不安な人は適当な文字列に変換しても良いと思います。
今回は小さなプライベートコミュニティなので何かあっても大丈夫かなと思っています。

# 最後に
終わってしまえば単純でしたが、MinIO の OpenID Configuration が壊れたりして結構苦労しました。
もし、正しいはずなのに動かないよ～って人は一度 OpenID Configuration を作り直したりすると良いかもしれません。
