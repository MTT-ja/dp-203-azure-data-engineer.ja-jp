---
lab:
  title: Azure Stream Analytics と Microsoft Power BI を使用してリアルタイム レポートを作成する
  ilt-use: Suggested demo
---

# Azure Stream Analytics と Microsoft Power BI を使用してリアルタイム レポートを作成する

データ分析ソリューションには、多くの場合、データ "ストリーム" を取り込んで処理するための要件が含まれます。** ストリーム処理は、通常は "無限" であるという点でバッチ処理とは異なります。つまり、ストリームは、一定の間隔ではなく永続的に処理する必要がある連続したデータ ソースです。**

Azure Stream Analytics には、ストリーミング ソースからのデータ ストリーム (Azure Event Hubs や Azure IoT Hub など) を操作する "クエリ" を定義するために使用できるクラウド サービスが用意されています。** Azure Stream Analytics クエリを使用して、データ ストリームを処理し、リアルタイムの視覚化のために結果を Microsoft Power BI に直接送信することができます。

この演習では、Azure Stream Analytics を使用して、販売注文データのストリーム (オンライン小売アプリケーションから生成されるものなど) を処理します。 注文データは Azure Event Hubs に送信され、そこで Azure Stream Analytics ジョブによってデータの読み取りと集計が行われた後、Power BI に送信されます。そこで、レポートでデータを視覚化します。

この演習の所要時間は約 **45** 分です。

## 開始する前に

管理レベルのアクセス権を持つ [Azure サブスクリプション](https://azure.microsoft.com/free)が必要です。

Microsoft Power BI サービスへもアクセスできる必要があります。 これは、学校または組織から既に提供されている場合もありますが、[個人として Power BI サービスにサインアップ](https://learn.microsoft.com/power-bi/fundamentals/service-self-service-signup-for-power-bi)することも可能です。

## Azure リソースをプロビジョニングする

この演習では、データ レイク ストレージにアクセスできる Azure Synapse Analytics ワークスペースと、専用 SQL プールが必要です。 ストリーミング注文データを送信できる Azure Event Hubs 名前空間も必要です。

これらのリソースをプロビジョニングするために、PowerShell スクリプトと ARM テンプレートを組み合わせて使用します。

1. [Azure portal](https://portal.azure.com) (`https://portal.azure.com`) にサインインします。
2. ページ上部の検索バーの右側にある **[\>_]** ボタンを使用して、Azure portal に新しい Cloud Shell を作成します。メッセージが表示された場合は、***PowerShell*** 環境を選択して、ストレージを作成します。 次に示すように、Azure portal の下部にあるペインに、Cloud Shell のコマンド ライン インターフェイスが表示されます。

    ![クラウド シェル ペインを含む Azure portal のスクリーンショット。](./images/cloud-shell.png)

    > **注**: 前に *Bash* 環境を使ってクラウド シェルを作成している場合は、そのクラウド シェル ペインの左上にあるドロップダウン メニューを使って、***PowerShell*** に変更します。

3. ペインの上部にある区分線をドラッグして Cloud Shell のサイズを変更したり、ペインの右上にある **&#8212;** 、 **&#9723;** 、**X** アイコンを使用して、ペインを最小化または最大化したり、閉じたりすることができます。 Azure Cloud Shell の使い方について詳しくは、[Azure Cloud Shell のドキュメント](https://docs.microsoft.com/azure/cloud-shell/overview)をご覧ください。

4. PowerShell のペインで、次のコマンドを入力して、この演習を含むリポジトリを複製します。

    ```
    rm -r dp-203 -f
    git clone https://github.com/MicrosoftLearning/dp-203-azure-data-engineer dp-203
    ```

5. リポジトリが複製されたら、次のコマンドを入力してこの演習用のフォルダーに変更し、そこに含まれている **setup.ps1** スクリプトを実行します。

    ```
    cd dp-203/Allfiles/labs/19
    ./setup.ps1
    ```

6. メッセージが表示された場合は、使用するサブスクリプションを選択します (これは、複数の Azure サブスクリプションへのアクセス権を持っている場合にのみ行います)。

7. スクリプトの完了を待っている間に、次のタスクに進みます。

## Power BI ワークスペースを作成する

Power BI サービスで、"ワークスペース"** 内のデータセット、レポート、およびその他のリソースを整理します。 すべての Power BI ユーザーには、**My Workspace** という名前の既定のワークスペースがあり、この演習で使用することもできます。ただし、通常は、管理する個別のレポート ソリューションごとにワークスペースを作成することをお勧めします。

1. Power BI サービス資格情報を使用して、[https://app.powerbi.com/](https://app.powerbi.com/) で Power BI サービスにサインインします。
2. **[ワークスペース]** ページで、 **[ワークスペースの作成]** を選択します。

    ![Power BI の [ワークスペースの作成] タブのスクリーンショット。](./images/powerbi-create-workspace.png)

    > **注**: 試用版アカウントを使用している場合は、追加の試用版機能を有効にする必要がある場合があります。 

3. わかりやすい名前 (*mslearn-streaming* など) を使用して新しいワークスペースを作成します。

4. ワークスペースを表示したときに、ページ URL (`https://app.powerbi.com/groups/<GUID>/list` に類似) 内のグローバル一意識別子 (GUID) を書き留めます。 この GUID は後で必要になります。

## Azure Stream Analytics を使用してストリーミング データを処理する

Azure Stream Analytics ジョブでは、1 つ以上の入力からのストリーミング データを操作し、結果を 1 つ以上の出力に送信する永続的クエリを定義します。

### Stream Analytics のジョブの作成

1. Azure portal を含むブラウザー タブに戻り、スクリプトが完了したら、**db000-*xxxxxxx*** リソース グループがプロビジョニングされたリージョンを書き留めます。
2. Azure portal の**ホーム** ページで、 **[+ リソースの作成]** を選択し、`Stream Analytics job` を検索します。 次に、次のプロパティを使用して **Stream Analytics ジョブ**を作成します。
    - **サブスクリプション**:お使いの Azure サブスクリプション
    - **リソース グループ**: 既存の **dp203-*xxxxxxx*** リソース グループを選択します。
    - **名前**: `stream-orders`
    - **リージョン**: Synapse Analytics ワークスペースがプロビジョニングされているリージョンを選択します。
    - **ホスティング環境**: Cloud
    - **ストリーミング ユニット**: 1
3. デプロイが完了するのを待ってから、デプロイされた Stream Analytics ジョブ リソースに移動します。

### イベント データ ストリームの入力を作成する

1. **stream-orders** の概要ページで、 **[入力の追加]** を選択します。 次に、 **[入力]** ページで **[ストリーム入力の追加]** メニューを使用して、次のプロパティを含む**イベント ハブ**入力を追加します。
    - **入力エイリアス**: `orders`
    - **サブスクリプションからイベント ハブを選択する:** オン
    - **サブスクリプション**:お使いの Azure サブスクリプション
    - **イベント ハブ名前空間**: **events*xxxxxxx*** Event Hubs 名前空間を選択します
    - **イベント ハブ名**: 既存の **eventhub*xxxxxxx*** イベント ハブを選択します
    - **イベント ハブ コンシューマー グループ**: 既存の **$Default** コンシューマー グループを選択します
    - **認証モード**: システム割り当てマネージド ID を作成します
    - **パーティション キー**: "空白のままにします"**
    - **イベント シリアル化形式**: JSON
    - **[エンコード]**: UTF-8
2. 入力を保存し、作成されるまで待ちます。 いくつかの通知が表示されます。 「**接続テストが成功しました**」という通知を待ちます。

### Power BI ワークスペースの出力を作成する

1. **stream-orders** Stream Analytics ジョブの **[出力]** ページを表示します。 次に、 **[追加]** メニューを使用して、次のプロパティを含む **Power BI** 出力を追加します。
    - **出力エイリアス:** `powerbi-dataset`
    - **Power BI 設定を手動で選択する**: オン
    - **グループ ワークスペース**: "ワークスペースの GUID"**
    - **認証モード**: " **[ユーザー トークン]** を選択** し、下部にある **[承認]** ボタンを使用して**、Power BI アカウントにサインインします"**
    - **データセット名**: `realtime-data`
    - **テーブル名**: `orders`

2. 出力を保存し、作成されるまで待ちます。 いくつかの通知が表示されます。 「**接続テストが成功しました**」という通知を待ちます。

### イベント ストリームを集計するクエリを作成する

1. **stream-orders** Stream Analytics ジョブの **[クエリ]** ページを表示します。
2. 次のように、既定のクエリを変更します。

    ```
    SELECT
        DateAdd(second,-5,System.TimeStamp) AS StartTime,
        System.TimeStamp AS EndTime,
        ProductID,
        SUM(Quantity) AS Orders
    INTO
        [powerbi-dataset]
    FROM
        [orders] TIMESTAMP BY EventEnqueuedUtcTime
    GROUP BY ProductID, TumblingWindow(second, 5)
    HAVING COUNT(*) > 1
    ```

    このクエリで、**System.Timestamp** (**EventEnqueuedUtcTime** フィールドに基づく) を使用して、各製品 ID の合計数量が計算される各 5 秒の "タンブリング" (重複しないシーケンシャル) 枠の開始と終了が定義されることを確認します。**

3. クエリを保存します。

### ストリーミング ジョブを実行して注文データを処理する

1. **stream-orders** Stream Analytics ジョブの **[概要]** ページを表示し、 **[プロパティ]** タブでジョブの **[入力]** 、 **[クエリ]** 、 **[出力]** 、 **[関数]** を確認します。 **[入力]** と **[出力]** の数が 0 の場合は、 **[概要]** ページの **[&#8635;更新]** ボタンを使用して、**orders** 入力と **powerbi-dataset** 出力を表示します。
2. **[&#9655; 開始]** ボタンを選択し、ストリーミング ジョブを今すぐ開始します。 ストリーミング ジョブが正常に開始されたことを通知されるまで待ちます。
3. クラウド シェル ペインを再度開き、次のコマンドを実行して、100 件の注文を送信します。

    ```
    node ~/dp-203/Allfiles/labs/19/orderclient
    ```

4. 注文クライアント アプリの実行中に、Power BI アプリ ブラウザー タブに切り替えて、ワークスペースを表示します。
5. ワークスペースに **realtime-data** データセットが表示されるまで、Power BI アプリ ページを更新します。 このデータセットは、Azure Stream Analytics ジョブによって生成されます。

## Power BI でストリーミング データを視覚化する

ストリーミング注文データのデータセットが作成されたので、それを視覚的に表す Power BI ダッシュボードを作成できます。

1. ワークスペースの **[+ 新規]** ドロップダウン メニューで、 **[ダッシュボード]** を選択し、**注文の追跡**という名前の新しいダッシュボードを作成します。
2. **注文の追跡**ダッシュボードの **[&#9999;&#65039; 編集]** メニューで、 **[タイルの追加]** を選択します。 次に、 **[タイルの追加]** ペインで **[カスタム ストリーミング データ]** を選択し、 **[次へ]** をクリックします。

    ![Power BI の [タイルの追加] ペインのスクリーンショット。](./images/powerbi-stream-tile.png)

3. **[カスタム ストリーミング データ タイルの追加]** ペインの **[データセット]** で、**realtime-data** データセットを選択し、 **[次へ]** をクリックします。

4. 既定の視覚化の種類を **[折れ線グラフ]** に変更します。 次に、次のプロパティを設定し、 **[次へ]** をクリックします。
    - **軸**: EndTime
    - **値**: Orders
    - **表示する時間枠**: 1 分

5. **[タイルの詳細]** ペインで、 **[タイトル]** を **[リアルタイム注文数]** に設定し、 **[適用]** をクリックします。

6. Azure portal を含むブラウザー タブに戻り、必要に応じてクラウド シェル ペインを再度開きます。 次に、次のコマンドを再実行して、別の 100 件の注文を送信します。

    ```
    node ~/dp-203/Allfiles/labs/19/orderclient
    ```

7. 注文送信スクリプトの実行中に、**注文の追跡** Power BI ダッシュボードを含むブラウザー タブに戻り、Stream Analytics ジョブ (まだ実行中) による処理時に新しい注文データが反映されるように視覚化が更新されることを確認します。

    ![注文データのリアルタイム ストリームを示す Power BI レポートのスクリーンショット。](./images/powerbi-line-chart.png)

    **orderclient** スクリプトを再実行し、リアルタイム ダッシュボードでキャプチャされているデータを確認できます。

## リソースを削除する

Azure Stream Analytic および Power BI を調べ終わったら、不要な Azure コストを避けるために、作成したリソースを削除する必要があります。

1. Power BI レポートを含むブラウザー タブを閉じます。 次に、 **[ワークスペース]** ペインで、ワークスペースの **[&#8942;]** メニューにある **[ワークスペースの設定]** を選択し、ワークスペースを削除します。
2. Azure portal を含むブラウザー タブに戻って、クラウド シェル ペインを閉じ、 **[&#128454; 停止]** ボタンを使用して Stream Analytics ジョブを停止します。 Stream Analytics ジョブが正常に停止したことを示す通知が表示されるまで待ちます。
3. Azure portal の **[ホーム]** ページで、**[リソース グループ]** を選択します。
4. Azure イベント ハブと Stream Analytics リソースを含む **dp203-*xxxxxxx*** リソース グループを選択します。
5. リソース グループの **[概要]** ページの上部で、**[リソース グループの削除]** を選択します。
6. リソース グループ名として「**dp203-*xxxxxxx***」と入力し、これが削除対象であることを確認したら、 **[削除]** を選択します。

    数分後、この演習で作成されたリソースは削除されます。