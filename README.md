💰 連結会計システム / Consolidated Accounting System
Java / Spring Boot で構築した、実務直結の連結会計処理API

📌 プロジェクト概要
本プロジェクトは、連結会計における中核処理をJava/Spring Bootで実装した業務システムAPIです。

対象業務：

✅ 減価償却計算（定額法・定率法）

✅ 税効果会計（繰延税金資産・負債）

✅ アップストリーム取引の連結消去

✅ 少数株主持分の按分計算

特徴：

🔥 金融システム必須のBigDecimal高精度計算

🔥 会計仕訳をAPIが自動生成

🔥 3層アーキテクチャで保守性・拡張性MAX

📁 パッケージ構成
text
com.accounting
├── controller/    ← API受付口（JSON入出力）
├── service/       ← 会計計算エンジン（ビジネスロジック）
└── model/         ← データ構造（Request/Response）
3層完全分離で、「見てわかる」「直せる」「増やせる」設計。

🧱 1. モデル層（Model）：データの入れ物
📦 DepreciationRequest.java
java
public class DepreciationRequest {
    private BigDecimal cost;        // 取得原価（例：5,000,000円）
    private int usefulLife;        // 耐用年数（例：47年）
    private BigDecimal residualValue; // 残存価額（10% or 0円）
    private String method;         // 償却方法：STRAIGHT_LINE / DECLINING_BALANCE
}
✅ 最重要ルール

金額は 絶対にBigDecimal（float/double禁止）

1円単位の絶対精度を保証

getter/setter = SpringがJSONとJavaを自動変換する通路

⚙️ 2. サービス層（Service）：会計計算エンジン
📌 DepreciationService.java – 減価償却計算機
🔵 定額法（Straight-Line Method）
java
// 年間償却費 = (取得原価 - 残存価額) ÷ 耐用年数（四捨五入）
BigDecimal annualDepreciation = cost.subtract(residualValue)
    .divide(BigDecimal.valueOf(usefulLife), 2, RoundingMode.HALF_UP);
📐 計算ロジック

毎年同額を償却

円未満は四捨五入（会計基準準拠）

簿価が残存価額を下回らないよう制御

🔴 定率法（Declining-Balance Method）
java
// 償却率 = 1 - (残存価額/取得原価) ^ (1/耐用年数)
double rate = 1 - Math.pow(
    residualValue.divide(cost, 10, RoundingMode.HALF_UP).doubleValue(),
    1.0 / usefulLife
);
⚠️ 技術的限界

Math.pow() は double 必須 → ここだけやむを得ずdouble使用

ただし除算は BigDecimal で高精度計算済み

java
// 通常年：簿価 × 償却率（四捨五入）
annualDepreciation = bookValue.multiply(BigDecimal.valueOf(rate))
    .setScale(2, RoundingMode.HALF_UP);

// 最終年：簿価 - 残存価額（残り一括償却）
annualDepreciation = bookValue.subtract(residualValue);
📌 TaxEffectService.java – 税効果会計
java
// 繰延税金 = 一時差異 × 実効税率 ÷ 100
BigDecimal deferredTax = temporaryDifference
    .multiply(effectiveTaxRate)
    .divide(BigDecimal.valueOf(100), 2, RoundingMode.HALF_UP);
🎯 超重要：仕訳を自動生成
差異種別	仕訳	意味
ASSET（将来減算）	(借) 繰延税金資産 / (貸) 法人税等調整額	将来税金が減る（前払い）
LIABILITY（将来加算）	(借) 法人税等調整額 / (貸) 繰延税金負債	将来税金が増える（未払い）
✅ 連結決算で必須の税効果処理をAPIレスポンスとして仕訳出力

📌 UpstreamService.java – アップストリーム取引
java
// 未実現利益 = 取引金額 × 利益率 ÷ 100
BigDecimal unrealizedProfit = transactionAmount
    .multiply(profitMargin)
    .divide(BigDecimal.valueOf(100), 2, RoundingMode.HALF_UP);

// 連結側消去額 = 未実現利益 × 親会社持分比率 ÷ 100
BigDecimal consolidatedPortion = unrealizedProfit
    .multiply(ownershipPercentage)
    .divide(BigDecimal.valueOf(100), 2, RoundingMode.HALF_UP);

// 少数株主側負担額 = 未実現利益 - 連結側消去額
BigDecimal minorityPortion = unrealizedProfit.subtract(consolidatedPortion);
⚠️ アップストリーム（子→親）の核心

ダウンストリーム（親→子）は全額親会社負担

アップストリームは少数株主にも負担させる
→ 連結消去の本質ロジックを完全実装

📌 MinorityInterestService.java – 少数株主持分
java
// 少数株主持分 = 子会社利益 × 少数株主持分比率 ÷ 100
BigDecimal minorityInterest = subsidiaryProfit
    .multiply(minorityPercentage)
    .divide(BigDecimal.valueOf(100), 2, RoundingMode.HALF_UP);

// 親会社持分割合 = 子会社利益 - 少数株主持分
BigDecimal parentPortion = subsidiaryProfit.subtract(minorityInterest);
🎯 連結仕訳を自動生成
text
(借) 当期純利益 XXX円
    (貸) 親会社株主に帰属する当期純利益 XXX円
    (貸) 少数株主持分 XXX円
✅ 子会社利益を親会社と少数株主で「山分け」
✅ 連結財務諸表の本質をコード化

🎮 3. コントローラー層（Controller）：API受付口
📌 AccountingController.java
java
@RestController          // JSONを返すREST API
@RequestMapping("/api") // 全APIは /api 配下
@CrossOrigin(origins = "*") // フロントエンド別ドメインOK
public class AccountingController {

    @Autowired
    private DepreciationService depreciationService;  // DI: new不要！
    
    @PostMapping("/depreciation/calculate")
    public ResponseEntity<DepreciationResponse> calculateDepreciation(
            @RequestBody DepreciationRequest request) {
        return ResponseEntity.ok(depreciationService.calculate(request));
    }
}
✅ 依存性注入（@Autowired）

Springが自動でインスタンスを注入

new 書かない → メモリ効率UP・テスト容易性UP

✅ @RequestBody

JSON → Javaオブジェクトに自動変換

✅ ResponseEntity

HTTPステータスコードを制御（200 OK etc.）

📋 API一覧（全4エンドポイント）
メソッド	エンドポイント	リクエスト	レスポンス	処理内容
POST	/api/depreciation/calculate	DepreciationRequest	DepreciationResponse	減価償却計算（定額/定率）
POST	/api/tax-effect/calculate	TaxEffectRequest	TaxEffectResponse	税効果会計＋仕訳生成
POST	/api/upstream/calculate	UpstreamRequest	UpstreamResponse	アップストリーム消去計算
POST	/api/minority-interest/calculate	MinorityInterestRequest	MinorityInterestResponse	少数株主持分按分計算
✅ 全APIがPOST + JSON入出力
✅ 単一責任原則：1エンドポイント＝1会計処理

🎓 【総まとめ】このコードから学ぶべき4レベル
✅ レベル1：Java基礎文法（絶対必須）
コード	意味	使用箇所
package	クラスの住所	全ファイル
import	外部クラス利用宣言	全ファイル
public class	クラス定義	全ファイル
private 型 変数名	フィールド定義	Modelクラス
if/else	条件分岐	償却方法分岐
for	繰り返し	耐用年数ループ
new	インスタンス生成	Response作成
list.add()	リストに追加	償却スケジュール
list.get()	リストから取得	初年度償却費
✅ レベル2：Spring Boot必須アノテーション
アノテーション	意味	使用場所
@Service	ビジネスロジッククラス	Service層
@RestController	API受付クラス	Controller層
@Autowired	依存性注入（new不要）	Controllerフィールド
@PostMapping	POSTメソッド受付	Controllerメソッド
@RequestBody	JSON → Java変換	Controller引数
@CrossOrigin	CORS全許可	Controllerクラス
✅ レベル3：BigDecimalマスター（金融システム必須）
メソッド	意味	会計処理例
add()	足し算	取得原価 + 改良費
subtract()	引き算	取得原価 - 残存価額
multiply()	掛け算	未実現利益 × 持分比率
divide(値, 桁, 丸め)	割り算（桁指定）	償却費計算
compareTo()	比較	簿価 vs 残存価額
setScale()	小数点桁数指定	四捨五入
⚠️ 鉄則：金銭計算は全てBigDecimal。float/double禁止。

✅ レベル4：会計知識の実装ポイント（実務直結）
コード行	会計知識	実務での意味
cost.subtract(residualValue)	償却可能限度額	法定耐用年数で割るための前処理
.divide(..., 2, HALF_UP)	円未満四捨五入	1円単位の正確な償却費
if (year == usefulLife)	最終年調整	残存価額ピッタリに合わせる
"ASSET" / "LIABILITY"	一時差異の種類	仕訳の借方/貸方が逆転
unrealizedProfit - consolidatedPortion	少数株主負担	アップストリーム消去の核心
🚀 このプロジェクトの価値
観点	内容
🏢 実務直結	連結会計・税効果・持分法をコードで表現
💰 高精度計算	BigDecimalで1円単位の絶対精度
🧠 学習価値	Java初心者→Spring Boot実務レベルへ
🔧 保守性	3層アーキテクチャで見てわかる・直せる
📤 拡張性	IFRS16・減損・のれん償却にも対応可能
📌 今後の拡張予定
IFRS16（リース）対応

減損損失計算

のれん償却処理

在外子会社の換算処理

⭐ 最後に
「会計処理を、エンジニアの手に。」

本プロジェクトは、

会計知識ゼロのエンジニア → 実務ロジックを理解

会計士・経理職 → Java実装のイメージを獲得

初学者 → Spring Boot + 業務システムを体験

⭐ Star いただけると開発者が喜びます！
🔁 Fork / PR も大歓迎！
