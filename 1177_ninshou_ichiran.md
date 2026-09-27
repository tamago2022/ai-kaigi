# 1177号【ログイン・認証・鍵が要る先 全部の棚卸し】

2026-09-28 02:5x JST ／ 調べ方＝**Mac実機で実測**（心臓経由の一発コマンド2本。ブラウザ操作なし・0円）
実測の生ログ：`status/oneshot/done/n1177_kagi_shirabe.out` ／ `n1177b_kagi_shirabe2.out`

**この紙の約束：鍵の値は1バイトも書かない（名前と有無だけ）。分からないものは「不明」「未確認」と書く。**

---

## 0. 実測でわかった、今の全体像（3つ）

1. **鍵がしまえている先は8つだけ。**正本 `鍵の正本ファイル（Mac内・非公開）` に入っているのは
   Buffer / OpenAI / xAI / YouTube / Devin / Supabase(読み) の6系統（11行のうち3行はVITE_の複製）。
   **残りは全部「鍵が無い」か「ブラウザのcookieに頼っている」。**
2. **`gh` コマンドがMacに入っていない**（実測：`gh 無し`）。にもかかわらず `GitHub鍵の置き場（Mac内・非公開）`（40バイト・
   09-28 01:06更新）と keychain `gh:github.com` は在る。＝GitHubはトークンで叩けるが、CLIは無い。
3. **Chrome のcookieに頼っている口が tools/ に36本ある**（実測：`chrome|CDP|9222|cookie` で36ファイル）。
   X・Genspark・LINE・Lovableの一部がここに乗っている。**ここが「毎回ログインして」の発生源。**

---

## 1. 全部の棚（22行）

| サービス | 何に使っているか | 今の認証方式 | 期限 | 今まで何回切れたか（実測） | 切れたとき誰が直すか | たまごさんが押す回数（月） | 公式が長期トークンを出しているか | 詰まっている点 |
|---|---|---|---|---|---|---|---|---|
| **Buffer** | X/SNSへの予約投稿 | APIキー（OAuth2で発行したaccess token）／`BUFFER_ACCESS_TOKEN` **有り** | 公式に記載が見当たらない＝**不明** | 記録なし（09-28 00:53の実測は HTTP **429**＝回数制限。鍵切れではない） | 自動 | **0** | ○ ただし refresh token は**使い捨て**。再利用でグラント全体が失効 → [developers.buffer.com](https://developers.buffer.com/guides/authentication.html) | 24時間の回数制限に当たっている（429・reset待ち） |
| **OpenAI (ChatGPT API)** | 外部AIへの相談・下書き | APIキー／`OPENAI_API_KEY` **有り** | 消すまで無期限 | 0回 | 誰も（不要） | **0** | ○ [platform.openai.com](https://platform.openai.com/docs/api-reference/authentication) | **予算の栓が今日0円**。1175号で相談が投げられず0件 |
| **xAI (Grok)** | 外部AIへの相談 | APIキー／`XAI_API_KEY` **有り** | 消すまで無期限 | 0回（鍵は生きている） | 誰も（不要） | **0** | ○ [docs.x.ai](https://docs.x.ai/docs/overview) | **残高なしで403**。鍵ではなくお金 |
| **YouTube Data API** | 動画の数字取り | APIキー／`YOUTUBE_DATA_API_KEY` **有り** | 消すまで無期限 | 0回 | 誰も（不要） | **0** | ○ [developers.google.com](https://developers.google.com/youtube/v3/getting-started) | なし |
| **Devin** | 外注の実装 | APIキー／`DEVIN_API_KEY` **有り** | **不明**（公式ページで未確認） | 0回 | 誰も（不要） | **0** | **未確認** | GitHub Actions の課金停止でPRのCIが全落ち（09-28実測・Devin側の問題ではない） |
| **Supabase（Mac側・読み）** | 棚のデータを読む | 公開鍵（`SUPABASE_URL` / `PROJECT_ID` / `PUBLISHABLE_KEY`）**有り** | ダッシュボードで消すまで無期限 | 0回 | 誰も（不要） | **0** | ○ [supabase.com](https://supabase.com/docs/guides/getting-started/api-keys) | なし |
| **Supabase（書き込み鍵）** | 棚に書く・Edge Function | `SERVICE_ROLE_KEY` は**正本に無し**（`別プロジェクトの設定ファイル（非公開）` にだけ在る＝置き場が枝分かれ） | 無期限 | 不明 | 誰も（不要） | **0**（貼り直しが初回1回） | ○ 同上 | 正本に無いので `kagi.py` から読めない |
| **Supabase Secrets（本番）** | 本番Edge Functionが読む | Supabase側に保存 | 無期限 | 不明 | 誰も（不要） | **0** | ○ | **未実測**（Macに `supabase` CLI が無く、画面でしか見えない） |
| **GitHub** | コード・掲示板・公開 | トークン（`GitHub鍵の置き場（Mac内・非公開）` 40バイト・09-28 01:06更新／keychain `gh:github.com`）※`GITHUB_TOKEN` は正本に**無し** | fine-grained PAT で**無期限を選べる** | 0回 | 誰も（不要） | **0** | ○ [docs.github.com](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens) | **`gh` コマンドが未インストール**。API直叩きしか道が無い |
| **Anthropic（Claude CLI）** | 工場の本体 | keychain `Claude Code-credentials`（契約=**max**）。`expiresAt` の欄が**無い**＝`setup-token` の固定鍵 | **1年** | ログ実測：「ログインが切れ」**10回**／`no_launch`（発車停止）**10回**／`setup-token` **47回**。`auth_keeper.log` 469行中**389行**が切れ・失敗・401系 | 自動（`auth_keeper`）＋切れたら**人が1回** | **0**（年1回） | ○ 1年もつ固定鍵（`claude setup-token`） | 過去に「同時更新で鍵ごと消える」事故（2026-09-20 04:30）。固定鍵にして構造的に解決済み |
| **Lovable** | 本番の公開ボタン | OAuth2（`Lovable鍵の置き場（Mac内・非公開）`・09-28 00:59更新） | **8時間**（`expires_in=28800`） | `lovable_12h.log` は2行・切れ0回（機械が12時間ごとに巻き直している） | 自動（`lovable_12h.py`） | **0** | ○ 公式REST APIとAPIキーが存在。**キー自体の期限は公式に記載なし** → [docs.lovable.dev](https://docs.lovable.dev/integrations/lovable-api) | `LOVABLE_API_KEY` は正本に無し。8時間鍵を回し続ける形のまま |
| **Gmail（Google）** | 見張りの目（返信を拾う） | **鍵が無い**（`GMAIL_APP_PASSWORD` 無し・`Gmail鍵の置き場（Mac内・非公開）` 無し） | アプリパスワードは**無期限**（パスワード変更で失効） | — | 誰も（不要） | **初回1回**、以後0 | ○ [support.google.com](https://support.google.com/accounts/answer/185833) | **これが原因で見張り4本が赤**（LINE審査／LINE返信／Anthropic返信／外部窓口） |
| **Gemini** | 安い外部代行 | **鍵が無い**（keychain の `gemini` はMacアプリ用） | 消すまで無期限 | — | 誰も（不要） | **初回1回**、以後0 | ○ 無料で発行できる [aistudio.google.com](https://aistudio.google.com/apikey) | 無料で取れるのに置かれていないだけ |
| **X（Twitter）** | 投稿 | **鍵が無い**。コード内の `X_*` は全部ただの設定名（実測）。実体は**Chromeのcookie** | cookieなので**いつでも切れる** | 不明（台帳なし） | **人** | **?**（台帳が無い＝実測できていない） | △ OAuth1.0aの期限は公式に記載が見当たらない。★無料プランが廃止され**従量課金**（URL付き投稿 $0.200/件） | 直叩きすると1本約30円。Buffer経由に寄せれば0円 |
| **Instagram / Meta** | 投稿・取得 | **鍵が無い**（`META_ACCESS_TOKEN` / `IG_ACCESS_TOKEN` は名前だけ） | long-lived **60日**（`refresh_access_token` で機械が巻き直せる） | — | 自動（番人を置けば） | **初回1回**、以後0 | ○ [developers.facebook.com](https://developers.facebook.com/docs/instagram-platform/reference/refresh_access_token/) ※検索プレビュー経由・**再確認が要る** | アプリも鍵もまだ作っていない |
| **Spotify** | 曲データ | **鍵が無い**（`SPOTIFY_*` は名前だけ） | refresh token に**6ヶ月**の期限。refreshしても**リセットされない** | — | **半年に1回だけ人** | **年2回**（月0.17） | ×（公式が期限を入れた） [developer.spotify.com](https://developer.spotify.com/blog/2026-06-18-refresh-token-expiration) | 唯一「月0回」にできない先 |
| **fal.ai** | 画像・動画生成 | **鍵が無い** | 不明 | — | 誰も（不要） | **初回1回**、以後0 | **未確認** | コード内の読み口が**5つに割れている**（`FAL_KEY` / `FAL_API_KEY` / `FAL_AI_KEY` / `FAL_ADMIN_KEY` / `FAL_KEY_ID`） |
| **Notion** | 記録・共有 | **鍵が無い**（keychain `Notion Safe Storage` はデスクトップアプリ用） | 内部インテグレーショントークンは無期限 | — | 誰も（不要） | **初回1回**、以後0 | ○ [developers.notion.com](https://developers.notion.com/docs/authorization) | 鍵未発行 |
| **Obsidian** | Vault（脳） | **認証不要**（ローカルのファイル） | — | 0回 | — | **0** | 該当なし | なし |
| **LINE** | 店・返信 | **鍵が無い**。`LINE用Chromeプロファイル（Mac内・非公開）` ＝**Chromeプロファイル（cookie）に頼っている** | cookieなので**いつでも切れる** | 不明（台帳なし） | **人** | **?** | ○ Messaging API のチャネルアクセストークンは長期発行できる [developers.line.biz](https://developers.line.biz/en/docs/messaging-api/channel-access-tokens/) | cookie依存のまま。公式トークンに逃げていない |
| **Genspark** | 外部AIの実装代行 | **鍵が無い**。Chromeのcookie依存 | cookieなので**いつでも切れる** | 不明（台帳なし） | **人** | **?** | **未確認**（公式APIの有無を確認できていない） | 掲示板に書いても無反応（16時間コメント0・実測） |
| **Jules** | 外部AIの実装代行 | **自前の鍵なし**（GitHub連携で動く。`JULES_REPO` は設定名） | GitHub側の鍵に乗る | — | 誰も（不要） | **0** | ○（GitHubに乗るため） | なし |
| **Gumroad** | 販売 | `GUMROAD_ACCESS_TOKEN` はリポの `.env` に**在るのに** `kagi.py` から**読めない**（実測「無し」） | 無期限 | — | 誰も（不要） | **0** | ○ | **読み口が届いていない**（正本に写っていない） |
| **Chrome本体のプロファイル** | 上の cookie 系すべての土台 | keychain `Chrome Safe Storage` | 再起動・IP変化・プロファイル破損で**いつでも切れる** | 不明（台帳なし） | **人** | **?** | 該当なし。★Chrome 127以降の App-Bound Encryption で**取り出す道は塞がっている** | tools/ の**36本**がここに依存 |

---

## 2. 数の集計（実測）

- **認証・鍵が要る先＝22（Obsidianは認証不要なので実質21）**
- **公式の無期限（または機械が自動で巻き直せる）鍵がある＝○ 14**
  （Buffer・OpenAI・xAI・YouTube・Supabase読み・Supabase書き・Supabase Secrets・GitHub・Lovable・Gmail・Gemini・Meta/IG・Notion・Jules・Gumroad ※Supabase系3本を1つに数えると12）
- **切れる／人の手が要る＝× 6**（Anthropic 年1回・Spotify 半年1回・X・LINE・Genspark・Chromeプロファイル）
- **不明・未確認＝? 3**（Devin・fal.ai・Genspark の公式トークン有無）
- **鍵がまだ置かれていない＝7**（Gmail・Gemini・Meta/IG・Spotify・fal.ai・Notion・Supabase書き込み鍵）

### たまごさんが押す回数

**★実測の台帳が無い。**「月◯回」を数える仕組みが今まで無かったので、
cookie依存の4件（X・LINE・Genspark・Chrome）は **「?」としか書けない**。推測で埋めない。

数えられるものだけ：

| 種類 | 回数 |
|---|---|
| 機械が回している先（上の○）で押す回数 | **0回／月** |
| 初回だけ貼る（1回ずつ・以後0） | **合計7回** |
| 定期で必ず要るもの | Anthropic **年1回** ＋ Spotify **年2回** ＝ **月0.25回** |
| cookie依存の4件 | **実測できていない（?）** |

**次にやること（この紙の外）：押した回数を数える台帳を1本置く。数えていないものは減ったかどうかも言えない。**

---

## 3. ChatGPTにそのまま貼る相談文

> ここから下を丸ごとコピーして貼ってください。

---

あなたは大規模な個人自動化システムの設計者です。以下は、あるMac 1台の上で動いている
「AIの工場」が現在依存している認証・鍵の全棚卸しです（実測。値は伏せています）。

**問い：この構成で、人間（持ち主）のログイン操作を月0回にするには、何を変えればいいですか。
世界の定石（業界で標準的に採られている方法）で答えてください。**

条件：

- 持ち主は非エンジニア。ターミナルを打たせない。`.env` を触らせない。
- 追加の月額費用は極小にしたい（現状ほぼ0円運用）。
- macOS。Chrome 127以降の App-Bound Encryption により、Chromeプロファイルからの
  cookie取り出しは不可。
- 現在 `gh` CLI は未インストール。GitHub は API直叩きのみ。
- ブラウザ自動化（Playwright/Selenium）は、アカウント停止リスクがあるため最終手段。

特に答えてほしい4点：

1. **cookie依存の4件（X・LINE・Genspark・Chromeプロファイル）を、公式の長期トークンに
   逃がす具体的な手順**。逃げられないものはどれで、なぜか。
2. **鍵の置き場**。現状は `鍵の正本ファイル（Mac内・非公開）`(600) 1本＋予備6か所という自前運用。
   1Password CLI / macOS keychain / age暗号 などに寄せるべきか、自前で十分か。
   判断の分かれ目は何か。
3. **「切れる前に機械が気づく」仕組みの定石**。各サービスの期限をどう台帳化し、
   何日前に何をするのが標準か。
4. **押した回数の計測**。人間の手作業回数をどう計測するのが定石か
   （現状、計測の仕組みが無いため「月◯回」を実測できていない）。

以下が棚卸し表です。

（★ここに上の「1. 全部の棚（22行）」の表を貼る）

---

## 4. この紙の限界（正直に）

- **「今まで何回切れたか」は、ログが残っている先しか数えられない。**
  Anthropic だけが `auth_keeper.log` を持っているので数えられた（10回）。
  X・LINE・Genspark・Chrome は**台帳が無いので0回とも100回とも書けない**。
- **Devin・fal.ai・Genspark の公式トークン期限は、公式ページで確認できていない**（「未確認」）。
- **Supabase Secrets（本番側）は実測していない**（Macに `supabase` CLI が無い）。
- Meta/Instagram の60日自動更新は、検索プレビュー経由で読んだもの。**公式ページの直読みで再確認が要る。**
