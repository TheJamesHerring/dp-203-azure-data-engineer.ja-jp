---
lab:
  title: Azure Synapse Analytics でデータ パイプラインを構築する
  ilt-use: Lab
---

# Azure Synapse Analytics でデータ パイプラインを構築する

この演習では、Azure Synapse Analytics Explorer のパイプラインを使用して専用 SQL プールにデータを読み込みます。 パイプラインによって、製品データをデータ ウェアハウス内のテーブルに読み込むデータ フローがカプセル化されます。

この演習の所要時間は約 **45** 分です。

## 開始する前に

管理レベルのアクセス権を持つ [Azure サブスクリプション](https://azure.microsoft.com/free)が必要です。

## Azure Synapse Analytics ワークスペースをプロビジョニングする

データ レイク ストレージにアクセスできる Azure Synapse Analytics ワークスペースと、リレーショナル データ ウェアハウスをホストする専用 SQL プールが必要です。

この演習では、Azure Synapse Analytics ワークスペースをプロビジョニングするために、PowerShell スクリプトと ARM テンプレートを組み合わせて使用します。

1. [Azure portal](https://portal.azure.com) (`https://portal.azure.com`) にサインインします。
2. ページ上部の検索バーの右側にある **[\>_]** ボタンを使用して、Azure portal に新しい Cloud Shell を作成します。メッセージが表示された場合は、***PowerShell*** 環境を選択して、ストレージを作成します。 次に示すように、Azure portal の下部にあるペインに、Cloud Shell のコマンド ライン インターフェイスが表示されます。

    ![Azure portal と Cloud Shell のペイン](./images/cloud-shell.png)

    > **注**: 前に *Bash* 環境を使ってクラウド シェルを作成している場合は、そのクラウド シェル ペインの左上にあるドロップダウン メニューを使って、***PowerShell*** に変更します。

3. ペインの上部にある区分線をドラッグして Cloud Shell のサイズを変更でき、ペインの右上にある —、 **&#9723;** 、**X** のアイコンを使用してペインを最小化、最大化、または閉じることができます。 Azure Cloud Shell の使い方について詳しくは、[Azure Cloud Shell のドキュメント](https://docs.microsoft.com/azure/cloud-shell/overview)をご覧ください。

4. PowerShell のペインで、次のコマンドを入力して、このリポジトリを複製します。

    ```powershell
    rm -r dp-203 -f
    git clone https://github.com/MicrosoftLearning/dp-203-azure-data-engineer dp-203
    ```

5. リポジトリが複製されたら、次のコマンドを入力してこの演習用のフォルダーに変更し、そこに含まれている **setup.ps1** スクリプトを実行します。

    ```powershell
    cd dp-203/Allfiles/labs/10
    ./setup.ps1
    ```

6. メッセージが表示された場合は、使用するサブスクリプションを選択します (これは、複数の Azure サブスクリプションへのアクセス権を持っている場合にのみ行います)。
7. メッセージが表示されたら、Azure Synapse SQL プールに設定する適切なパスワードを入力します。

    > **注**: このパスワードは忘れないようにしてください。

8. スクリプトの完了まで待ちます。通常、約 10 分かかりますが、さらに時間がかかる場合もあります。 待っている間、Azure Synapse Analytics ドキュメントの「[Azure Synapse Analytics のデータ フロー](https://learn.microsoft.com/azure/synapse-analytics/concepts-data-flow-overview)」の記事を確認してください。

## ソースとターゲットのデータ ストアを表示する

この演習のソース データは、製品データを含むテキスト ファイルです。 宛先は、専用 SQL プール内のテーブルです。 目標は、ファイル内の製品データがテーブルに読み込まれるデータ フローをカプセル化するパイプラインを作成することです。新しい製品が挿入され、既存のものは更新されます。

1. スクリプトが完了したら、Azure portal で、作成された **dp203-*xxxxxxx*** リソース グループに移動して、Synapse ワークスペースを選択します。
2. Synapse ワークスペースの **[概要]** ページの **[Synapse Studio を開く]** カードで **[開く]** を選択し、新しいブラウザー タブで Synapse Studio を開きます。ダイアログが表示された場合はサインインします。
3. Synapse Studio の左側にある ›› アイコンを使用してメニューを展開します。これにより、リソースの管理とデータ分析タスクの実行に使用するさまざまなページが Synapse Studio 内に表示されます。
4. **[管理]** ページの **[SQL プール]** タブで、専用 SQL プール **sql*xxxxxxx*** の行を選択し、 **&#9655;** アイコンを使用して起動します。ダイアログが表示されたら、再開することを確認します。

     プールの再開には数分かかる場合があります。 **&#8635; [最新の情報に更新]** ボタンを使用して、その状態を定期的に確認できます。 準備ができると、状態が **[オンライン]** と表示されます。 待っている間に、次のステップに進み、ソース データを表示します。

5. **[データ]** ページで **[リンク]** タブを表示して、**synapse*xxxxxxx* (Primary - datalake*xxxxxxx*)** のような名前の Azure Data Lake Storage Gen2 ストレージ アカウントへのリンクがワークスペースに含まれていることを確認します。
6. ストレージ アカウントを展開して、そこに **files (primary)** という名前のファイル システム コンテナーが含まれていることを確認します。
7. このファイル コンテナーを選択し、そこに **data** という名前のフォルダーが含まれていることに注目します。
8. **[data]** フォルダーを開き、そこに含まれている **[Product.csv]** ファイルを確認します。
9. **[Product.csv]** を右クリックし、 **[プレビュー]** を選択して、そこに含まれているデータを表示します。 ヘッダー行と製品データの一部のレコードが含まれていることに注目してください。
10. **[管理]** ページに戻り、専用 SQL プールがオンラインになったことを確認します。 そうでない場合は、お待ちください。
11. **[データ]** ページの **[ワークスペース]** タブで、**SQL データベース**、**sql*xxxxxxx* (SQL)** データベース、およびその**テーブル**を展開します。
12. **dbo.DimProduct** テーブルを選択します。 次に、その **[...]** メニューで、 **[新しい SQL スクリプト]**  >  **[上位 100 行を選択]** を選択します。テーブルから製品データを返すクエリが実行されます。1 行になるはずです。

## パイプラインを実装する

テキスト ファイル内のデータをデータベース テーブルに読み込むには、テキスト ファイルからデータを取り込むロジックをカプセル化するデータフローを含む Azure Synapse Analytics パイプラインを実装し、データベースに既に存在する製品の代理 **[ProductKey]** 列を検索し、それに応じてテーブル内の行を挿入または更新します。

### データ フロー アクティビティが含まれるパイプラインの作成

1. Synapse Studio で、 **[統合]** ページを選択します。 次に、 **+** メニューで、 **[パイプライン]** を選択して新しいパイプラインを作成します。
2. 新しいパイプラインの **[プロパティ]** ペインで、その名前を **[Pipeline1]** から **[製品データの読み込み]** に変更します。 次に、 **[プロパティ]** ペインの上にある **[プロパティ]** ボタンを使用して、それを非表示にします。
3. **[アクティビティ]** ペインで、 **[移動と変換]** を展開します。次に示すように、 **[データ フロー]** をパイプライン設計画面にドラッグします。

    ![データ フロー アクティビティが含まれるパイプラインのスクリーンショット。](./images/dataflow.png)

4. パイプライン設計画面の **[全般]** タブで、 **[名前]** プロパティを **[LoadProducts]** に設定します。
5. **[設定]** タブの設定の一覧の下部にある **[ステージング]** を展開し、次のステージング設定を設定します。
    - **[ステージングのリンク サービス]** : **synapse*xxxxxxx*-WorkspaceDefaultStorage** リンク サービスを選択します。
    - **[ステージング ストレージ フォルダー]** : **[コンテナー]** を **[ファイル]** に、 **[ディレクトリ]** を **[stage_products]** に設定します。

### データ フローを構成する

1. **[LoadProducts]** データ フローの **[設定]** タブの上部にある **[データ フロー]** プロパティで、 **[+ 新規]** を選択します。
2. 新しく開いたデータ フロー設計画面の **[プロパティ]** ペインで、 **[名前]** を **[製品データの読み込み]** に設定し、 **[プロパティ]** ペインを非表示にします。 データ フロー デザイナーは次のようになります。

    ![空のデータ フロー アクティビティのスクリーンショット。](./images/empty-dataflow.png)

### ソースを追加する

1. データ フロー設計画面の **[ソースの追加]** ドロップダウン リストで、 **[ソースの追加]** を選択します。 次に、ソース設定を次のように構成します。
    - **[出力ストリーム名]** : ProductsText
    - **[説明]** : 製品のテキスト データ
    - **[ソースの種類]** : 統合データセット
    - **[データセット]** : 次のプロパティで**新しい**データセットを追加します。
        - **[種類]** : Azure DataLake Storage Gen2
        - **[形式]** : 区切りテキスト
        - **[名前]** : Products_Csv
        - **[リンク サービス]** : synapse*xxxxxxx*-WorkspaceDefaultStorage
        - **[ファイル パス]** : files/data/Product.csv
        - **最初の行をヘッダーとして使用**: 選択
        - **スキーマのインポート**:接続/ストアから
    - **[スキーマの誤差を許可]** : 選択
2. 新しい **[ProductsText]** ソースの **[プロジェクション]** タブで、次のデータ型を設定します。
    - **[ProductID]** : 文字列
    - **[ProductName]** : 文字列
    - **[Color]** : 文字列
    - **[Size]** : 文字列
    - **[ListPrice]** : 10 進
    - **[Discontinued]** : ブール値
3. 次のプロパティを持つ 2 番目のソースを追加します。
    - **[出力ストリーム名]** : ProductsTable
    - **[説明]** : 製品テーブル
    - **[ソースの種類]** : 統合データセット
    - **[データセット]** : 次のプロパティで**新しい**データセットを追加します。
        - **[種類]** : Azure Synapse Analytics
        - **[名前]** : DimProduct
        - **[リンク サービス]** : 次のプロパティを使用して**新しい**リンク サービスを作成します。
            - **[名前]** : Data_Warehouse
            - **[説明]** : 専用 SQL プール
            - **統合ランタイム経由で接続する**: AutoResolveIntegrationRuntime
            - **[アカウントの選択方法]** : Azure サブスクリプションから
            - **[Azure サブスクリプション]** : Azure サブスクリプションを選択します
            - **[サーバー名]** : synapse*xxxxxxx* (Synapse ワークスペース)
            - **[データベース名]** : synapse*xxxxxxx*
            - **[SQL プール]** : synapse**
             **[認証の種類]** : システム割り当てマネージド ID
        - **[テーブル名]** : dbo.DimProduct
        - **スキーマのインポート**:接続/ストアから
    - **[スキーマの誤差を許可]** : 選択
4. 新しい **ProductTable** ソースの **[プロジェクション]** タブで、次のデータ型が設定されていることを確認します。
    - **[ProductKey]** : 整数
    - **[ProductAltKey]** : 文字列
    - **[ProductName]** : 文字列
    - **[Color]** : 文字列
    - **[Size]** : 文字列
    - **[ListPrice]** : 10 進
    - **[Discontinued]** : ブール値
5. 次に示すように、データ フローに 2 つのソースが含まれていることを確認します。

    ![2 つのソースを含むデータ フローのスクリーンショット。](./images/dataflow_sources.png)

### 検索を追加する

1. **[ProductsText]** ソースの右下にある **+** アイコンを選択し、 **[検索]** を選択します。
2. 検索の設定を次のように構成します。
    - **[出力ストリーム名]** : MatchedProducts
    - **[説明]** : 一致する製品データ
    - **[プライマリ ストリーム]** : ProductText
    - **[検索ストリーム]** : ProductTable
    - **[Match multiple rows](複数の行の一致)** : 選択<u>なし</u>
    - **[一致対象]** : 最終行
    - **[並べ替え条件]** : ProductKey 昇順
    - **[検索条件]** : ProductID == ProductAltKey
3. データ フローが次のようになっていることを確認します。

    ![2 つのソースと検索を含むデータ フローのスクリーンショット。](./images/dataflow_lookup.png)

    この検索では "両方" のソースから一連の列が返され、基本的に、テキスト ファイルの **[ProductID]** 列とデータ ウェアハウス テーブルの **[ProductAltKey]** 列が一致する外部結合が形成されます。** 代替キーを持つ製品がテーブルに既に存在する場合、データセットには両方のソースの値が含まれます。 製品がまだデータ ウェアハウスに存在しない場合、データセットにはテーブル列の NULL 値が含まれます。

### 行の変更を追加する

1. **[MatchedProducts]** 検索の右下にある **+** アイコンを選択し、 **[行の変更]** を選択します。
2. 行の変更設定を次のように構成します。
    - **[出力ストリーム名]** : SetLoadAction
    - **[説明]** : 新規挿入、既存のアップサート
    - **[受信ストリーム]** : MatchedProducts
    - **[行の変更条件]** : 既存の条件を編集し、 **+** ボタンを使用して次のように 2 つ目の条件を追加します (式では "大文字と小文字が区別される" ことに注意してください)。**
        - InsertIf: `isNull(ProductKey)`
        - UpsertIf: `not(isNull(ProductKey))`
3. データ フローが次のようになっていることを確認します。

    ![2 つのソース、検索、行の変更を含むデータ フローのスクリーンショット。](./images/dataflow_alterrow.png)

    行の変更ステップでは、各行に対して実行する読み込みアクションの種類を構成します。 テーブルに既存の行がない ( **[ProductKey]** が NULL である) 場合、テキスト ファイルの行が挿入されます。 製品の行が既にある場合は、既存の行を更新するために "アップサート" が実行されます。** この構成では、基本的に "タイプ 1 の緩やかに変化するディメンションの更新" が適用されます。**

### シンクを追加する

1. **SetLoadAction** の行の変更ステップの右下にある **+** アイコンを選択し、 **[シンク]** を選択します。
2. **[シンク]** プロパティを次のように構成します。
    - **[出力ストリーム名]** : DimProductsTable
    - **[説明]** : DimProduct テーブルを読み込む
    - **[受信ストリーム]** : SetLoadAction
    - **[シンクの種類]** : 統合データセット
    - **[データセット]** : DimProduct
    - **[スキーマの誤差を許可]** : 選択
3. 新しい **[DimProductTable]** シンクの **[設定]** タブで、次の設定を指定します。
    - **[更新方法]** : **[挿入を許可]** と **[アップサートを許可]** を選択します。
    - **[キー列]** : **[列の一覧]** を選択し、 **[ProductAltKey]** 列を選択します。
4. 新しい **[DimProductTable]** シンクの **[マッピング]** タブで、 **[自動マッピング]** チェック ボックスをオフにし、次の列マッピング "のみ" を指定します。<u></u>
    - ProductID: ProductAltKey
    - ProductsText@ProductName: ProductName
    - ProductsText@Color: Color
    - ProductsText@Size: サイズ
    - ProductsText@ListPrice: ListPrice
    - ProductsText@Discontinued: Discontinued
5. データ フローが次のようになっていることを確認します。

    ![2 つのソース、検索、行の変更、シンクを含むデータ フローのスクリーンショット。](./images/dataflow-sink.png)

## データ フローをデバッグする

パイプラインにデータ フローを構築したので、これを発行する前にデバッグできます。

1. データ フロー デザイナーの上部で、**データ フローのデバッグ**を有効にしました。 既定の構成を確認し、 **[OK]** を選択し、デバッグ クラスターが起動するまで待ちます (数分かかる場合があります)。
2. データ フロー デザイナーで、 **[DimProductTable]** シンクを選択し、その **[データ プレビュー]** タブを表示します。
3. **&#8635; [最新の情報に更新]** ボタンを使用してプレビューを更新します。これには、データ フローを介してデータを実行してデバッグする効果があります。
4. プレビュー データを確認し、1 つのアップサートされた行 (既存の *AR5381* 製品の場合) を **<sub>*</sub><sup>+</sup>** アイコンで、挿入された 10 行を **+** アイコンで示しています。

## パイプラインを発行して実行する

パイプラインを発行して実行する準備ができました。

1. **[すべて発行]** ボタンを使用して、パイプライン (およびその他の未保存の資産) を発行します。
2. 発行が完了したら、 **[LoadProductsData]** データ フロー ペインを閉じて、 **[製品データの読み込み]** パイプライン ペインに戻ります。
3. パイプライン デザイナー ペインの上部にある **[トリガーの追加]** メニューで、 **[今すぐトリガー]** を選択します。 次に、 **[OK]** を選択して、パイプラインの実行を確定します。

    **注**: スケジュールされた時刻または特定のイベントに応答してパイプラインを実行するトリガーを作成することもできます。

4. パイプラインの実行が開始されたら、 **[監視]** ページで **[パイプラインの実行]** タブを表示し、 **[製品データの読み込み]** パイプラインの状態を確認します。

    パイプラインの完了には 5 分以上かかる場合があります。 ツール バーの **&#8635; [最新の情報に更新]** ボタンを使用して、その状態を確認できます。

5. パイプラインの実行が成功したら、 **[データ]** ページで SQL データベースの **[dbo.DimProduct]** テーブルの **[...]** メニューを使用して、上位 100 行を選択するクエリを実行します。 テーブルに、パイプラインによって読み込まれたデータが含まれているはずです。
   
## Azure リソースを削除する

Azure Synapse Analytics を調べ終わったら、不要な Azure コストを避けるために、作成したリソースを削除する必要があります。

1. Synapse Studio ブラウザー タブを閉じ、Azure portal に戻ります。
2. Azure portal の **[ホーム]** ページで、**[リソース グループ]** を選択します。
3. Synapse Analytics ワークスペースに対して **dp203-*xxxxxxx*** リソース グループ (管理対象リソース グループ以外) を選択し、そこに Synapse ワークスペース、ストレージ アカウント、ワークスペースの専用 SQL プールが含まれていることを確認します。
4. リソース グループの **[概要]** ページの上部で、**[リソース グループの削除]** を選択します。
5. リソース グループ名「**dp203-*xxxxxxx***」を入力してそれを削除することを確認し、 **[削除]** を選びます。

    数分後に、Azure Synapse ワークスペース リソース グループと、それに関連付けられているマネージド ワークスペース リソース グループが削除されます。