# Azure Cosmos DB Change Feed

このラボでは、Change Feed Processor Library と Azure Functions を使用して、Azure Cosmos DB Change Feed の 3 つのユースケースを実装します。

> これが初めてのラボで、ラボの内容のセットアップがまだ完了していない場合は、このラボを開始する前に、[アカウント セットアップ](00-account_setup.md) の説明を参照してください。

## Build A .NET Console App to Generate Data

Eコマースサイトのアクションという形で、店舗に流れ込むデータをシミュレートするために、Cosmos DB CartContainerにドキュメントを生成して追加する簡単な.NETコンソールアプリを構築します。

1. ローカル マシンで、[Documents] フォルダにある [CosmosLabs] フォルダを探し、.NET Core プロジェクトのコンテンツを格納するために使用される `Lab08` フォルダを開きます。Microsoft Hands-on Labs を使用してこのラボを完了する場合、CosmosLabs フォルダは次のパスにあります： **C:∕CosmosLabs∕CosmosLabs∕CosmosLabs

1. Lab08`フォルダでフォルダを右クリックし、**Open with Code** メニューオプションを選択します。

   > あるいは、カレントディレクトリでターミナルを起動し、`code .` コマンドを実行する。

1. 左側のエクスプローラーペインで、**DataGenerator** フォルダーを探し、展開します。

1. Explorer**ペインで`program.cs`リンクを選択し、エディターでファイルを開きます。

   ![program.csが表示されます](../media/08-console-main-default.jpg "program.csファイルを開く")

1. 変数 `_endpointUrl` はプレースホルダーの値を **URI** 値に、変数 `_primaryKey` はプレースホルダーの値を Azure Cosmos DB アカウントの **PRIMARY KEY** 値に置き換えます。これらの値をまだ持っていない場合は、[これらの説明](00-account_setup.md) を使用してこれらの値を取得してください：

   - 例えば、**url** が `https://cosmosacct.documents.azure.com:443/` の場合、新しい変数の代入は次のようになります：

   ```csharp
   private static readonly string _endpointUrl = "https://cosmosacct.documents.azure.com:443/";
   ```

   - 例えば、**プライマリキー**が`elzirrKCnXlacvh1CRAnQdYVbVLspmYHQyrhx0PltHi8wn5lHVHFnd1Xm3ad5cn4TUcH4U0MSeHsVykFPHpQ===`の場合、新しい変数の代入は次のようになります：

   ```csharp
   private static readonly string _primaryKey = "elzirrKCnXlacvh1CRAnQdYVbVLspmYHQyYrhx0PltHi8wn5lHVHFnd1Xm3ad5cn4TUcH4U0MSeHsVykkFPHpQ==";
   ```

### Cosmos DBにドキュメントを追加する関数の作成

コンソールアプリケーションの主な機能は、Cosmos DBにドキュメントを追加して、eコマースウェブサイトのアクティビティをシミュレートすることです。ここでは、ドキュメントのデータ定義を作成し、ドキュメントを追加する関数を定義します。

1. DataGenerator**フォルダの `program.cs` ファイルで、`AddItem()` メソッドを探します。このメソッドの目的は、CosmosDBコンテナに**CartAction**のインスタンスを追加することです。

   > CosmosDBコンテナにドキュメントを追加する方法を復習したい場合は、[ラボ01 ](01-creating_partitioned_collection.md)を参照してください。

### 買い物データをランダムに生成する関数の作成

1. DataGenerator**フォルダ内の`Program.cs`ファイル内で、`GenerateActions()`メソッドを探します。このメソッドの目的は、CosmosDBの変更フィードを使用して消費するランダムな **CartAction** オブジェクトを作成することです。

### コンソールアプリの実行と機能の確認

このステップでは、Cosmos DB アカウントを見て、テストデータが期待通りに書き込まれていることを確認します。

1. ターミナルウィンドウを開きます。
2. ターミナルペインで、以下のコマンドを入力して実行し、コンソールアプリを実行します：

   ```sh
   cd DataGenerator

   dotnet run
   ```

3. 短いビルドプロセスの後、データが生成されCosmos DBに書き込まれると、アスタリスクが表示されます。

   ターミナル・ウインドウが表示され、アスタリスクが出力される](../media/08-console-running.jpg "プログラムを実行し、1、2分走らせる。")

4. コンソールアプリを1～2分実行させ、コンソールのいずれかのキーを押して停止させる。

5. AzureポータルとCosmos DBアカウントに切り替えます。

6. 6. **Azure Cosmos DB** ブレードから、左側の **Data Explorer** タブを選択します。

   データエクスプローラーがハイライトされたCosmos DBリソース](../media/08-cosmos-overview-final.jpg "Open the Data Explorer")

7. StoreDatabase**を展開し、次に**CartContainer**を展開し、**Items**を選択します。以下のスクリーンショットのようなものが表示されるはずです。

   > 重要なことは、ここにデータがあるということです。

   ![StoreDatabaseのアイテムが選択されています](../media/08-cosmos-data-explorer-with-data.jpg "Select an item in the StoreDatabase")

## Change Feed ProcessorによるCosmos DB Change Feedの利用

Cosmos DBの変更フィードを利用するには、Azure FunctionsとChange Feed Processorライブラリの2つの方法があります。まずは、Change Feed Processor をシンプルなコンソールアプリケーションで使ってみましょう。

### Cosmos DB Change Feedへの接続

Cosmos DB Change Feedの最初のユースケースはライブマイグレーションです。Cosmos DB コンテナを設計する際によくある懸念は、パーティションキーを適切に選択することです。パーティションキーを `/Item` として `CartContainer` を作成したことを思い出してください。このキーが間違っていることに後で気づいたらどうしよう。あるいは、書き込みは `/Item` の方がうまくいき、読み込みは `/BuyerState` をパーティションキーにした方がうまくいくとしたらどうでしょう？Cosmos DB Change Feedを使用して、異なるパーティションキーを持つ2つ目のコンテナにリアルタイムでデータを移行することで、分析麻痺を回避できる！

1. Visual Studio Codeに戻る

2. Explorer**ペインの**ChangeFeedConsole**フォルダの下にある`Program.cs`リンクを選択し、エディタでファイルを開きます。

3. 3. 変数 `_endpointUrl` のプレースホルダの値を **URI** 値に置き換え、変数 `_primaryKey` のプレースホルダの値を Azure Cosmos DB アカウントの **PRIMARY KEY** 値に置き換えます。

4. program.cs` ファイルの一番上にあるコンテナ構成値の、`_containerId` に続くコピー先コンテナ名に注目する：

   ```csharp
   private static readonly string _destinationContainerId = "CartContainerByState";
   ```

   > この場合、同じデータベース内の別のコンテナにデータを移行します。データをまったく別のデータベースに移行する場合でも、同じ考え方が適用されます。

5. 変更フィードを利用するために、**リースコンテナ** を使用します。以下のコードを `//todo: Add lab code here` の代わりに追加して、リースコンテナを作成します：

   ```csharp
   ContainerProperties leaseContainerProperties = new ContainerProperties("consoleLeases", "/id");
   Container leaseContainer = await database.CreateContainerIfNotExistsAsync(leaseContainerProperties, throughput: 400);
   ```

   > リースコンテナ**は、変更フィードの並列処理を可能にするための情報を格納し、フィードからの変更を最後に処理した場所のブックマークとして機能します。

6. ここで、変更プロセッサのインスタンスを取得するために、 **leaseContainer** 定義の直後に以下のコードを追加します：

   ```csharp
   var builder = container.GetChangeFeedProcessorBuilder("migrationProcessor", (IReadOnlyCollection<object> input, CancellationToken cancellationToken) => {
       Console.WriteLine(input.Count + " Changes Received");
       //todo: Add processor code here
   });

   var processor = builder
                   .WithInstanceName("changeFeedConsole")
                   .WithLeaseContainer(leaseContainer)
                   .Build();
   ```

   > 変更のセットを受信するたびに、`CreateChangeFeedProcessorBuilder` で定義された `Func<T>` が呼び出されます。この変更点の処理は当面省略する。

7. プロセッサを実行するには、それを起動しなければならない。**processor**の定義に続いて、以下のコードを追加する：

   ```csharp
   await processor.StartAsync();
   ```

8. 最後に、キーが押されてプロセッサーが終了したら、それを終了させる必要がある。ここに停止コードを追加する`//todo: Add stop code here`の行を探し、次のコードに置き換える：

   ```csharp
   await processor.StopAsync();
   ```

9. この時点で、あなたの`Program.cs`ファイルは次のようになっているはずです：

   ```csharp
   using System;
   using System.Collections.Generic;
   using System.Threading;
   using System.Threading.Tasks;
   using Microsoft.Azure.Cosmos;

   namespace ChangeFeedConsole
   {
       class Program
       {
           private static readonly string _endpointUrl = "<your-endpoint-url>";
           private static readonly string _primaryKey = "<your-primary-key>";
           private static readonly string _databaseId = "StoreDatabase";
           private static readonly string _containerId = "CartContainer";
           private static readonly string _destinationContainerId = "CartContainerByState";
           private static CosmosClient _client = new CosmosClient(_endpointUrl, _primaryKey);

           static async Task Main(string[] args)
           {

               Database database = _client.GetDatabase(_databaseId);
               Container container = db.GetContainer(_containerId);
               Container destinationContainer = db.GetContainer(_destinationContainerId);

               ContainerProperties leaseContainerProperties = new ContainerProperties("consoleLeases", "/id");
               Container leaseContainer = await  db.CreateContainerIfNotExistsAsync(leaseContainerProperties, throughput: 400);

               var builder = container.GetChangeFeedProcessorBuilder("migrationProcessor",
                  (IReadOnlyCollection<object> input, CancellationToken cancellationToken) =>
                  {
                     Console.WriteLine(input.Count + " Changes Received");
                     //todo: Add processor code here
                  });

               var processor = builder
                               .WithInstanceName("changeFeedConsole")
                               .WithLeaseContainer(leaseContainer)
                               .Build();

               await processor.StartAsync();
               Console.WriteLine("Started Change Feed Processor");
               Console.WriteLine("Press any key to stop the processor...");

               Console.ReadKey();

               Console.WriteLine("Stopping Change Feed Processor");
               await processor.StopAsync();
           }
       }
   }
   ```

### ライブデータ移行の完了

1. ChangeFeedConsole**フォルダ内の`program.cs`ファイル内で、`//todo: Add processor code here`というTodoを探します。

1. GetChangeFeedProcessorBuilder` の `Func<T>` のシグネチャを以下のように変更し、 `object` を `CartAction` に置き換える：

   ```csharp
   var builder = container.GetChangeFeedProcessorBuilder("migrationProcessor", 
      (IReadOnlyCollection<CartAction> input, CancellationToken cancellationToken) =>
      {
         Console.WriteLine(input.Count + " Changes Received");
         //todo: Add processor code here
      });
   ```

1. input**は、変更された**CartAction**ドキュメントのコレクションです。これらをマイグレートするには、単純にこれらをループして、移行先のコンテナに書き出します。//todo:ここにプロセッサコードを追加する`を以下のコードに置き換える：

   ```csharp
   var tasks = new List<Task>();

   foreach (var doc in input)
   {
      tasks.Add(destinationContainer.CreateItemAsync(doc, new PartitionKey(doc.BuyerState)));
   }

   return Task.WhenAll(tasks);
   ```

### チェンジ・フィード機能の動作確認テスト

最初の Change Feed コンシューマーができたので、テストを実行し、それが機能することを確認します。

1. 2 番目の**ターミナル・ウィンドウを開き、**ChangeFeedConsole** フォルダに移動します。

1. 2 番目の**ターミナル・ウィンドウで以下のコマンドを実行して、コンソール・アプリを起動します：

   ```sh
   cd ChangeFeedConsole

   dotnet run
   ```

1. 関数の実行が始まると、コンソールに次のようなメッセージが表示される：

   ```sh
   Started Change Feed Processor
   Press any key to stop the processor...
   ```

   > このコンシューマーを実行するのは今回が初めてなので、消費するデータはない。変更を受け取り始めるために、データジェネレーターを開始する。

1. 最初の**ターミナル・ウィンドウで、**DataGenerator**フォルダに移動する。

1. 最初の**ターミナル・ウィンドウで以下のコマンドを実行して、**DataGenerator**を再度起動します。

   ```sh
   dotnet run
   ```

1. データが書き込まれるにつれて、アスタリスクが再び表示され始めるのがわかるはずだ。

1. データが書き込まれ始めるとすぐに、**2番目の**ターミナル・ウィンドウに以下の出力が表示され始める：

   ```sh
   100 Changes Received
   100 Changes Received
   3 Changes Received
   ...
   ```

1. 数分後、**cosmosdblab** データエクスプローラに移動し、**StoreDatabase** を展開し、次に **CartContainerByState** を展開し、**Items** を選択します。パーティションキーが`/BuyerState`であることに注意してください。

   ![状態別カートコンテナが表示されます](../media/08-cart-container-by-state.jpg "CartContainerByState を開いてアイテムを確認する")

1. 最初の**端末のいずれかのキーを押して、データ生成を停止する。

1. ChangeFeedConsole**の実行を終了させてください(それほど時間はかからないはずです)。新しいログメッセージの書き込みが止まれば、終了したことがわかる。番目の**ターミナル・ウィンドウでどれかのキーを押して、関数を停止してください。

> これで、ライブデータを新しいコレクションに書き込む最初のCosmos DB Change Feedコンシューマが作成されました。おめでとうございます！次のステップでは、Azure Functionsを使ってCosmos DBの変更フィードを利用する2つのユースケースを紹介します。

## Cosmos DBの変更フィードを利用するAzure関数の作成

Azure Cosmos DBの興味深い機能の1つに、変更フィードがあります。変更フィードは多くのシナリオをサポートしていますが、このラボではそのうちの3つをさらに調査します。

### Create a .NET Core Azure Functions Project

この演習では、Azure Cosmos DB の変更フィードをスケーラブルかつフォールトトレラントな方法で読み込むために、.NET SDK の変更フィードプロセッサーライブラリを実装します。Azure Functions は、変更フィードプロセッサを実装することで、Cosmos DB の変更フィードにすばやく簡単に接続できます。まず、.NET Core Azure Functionsプロジェクトをセットアップします。

> 詳細は[doc](https://docs.microsoft.com/azure/cosmos-db/sql/change-feed-processor)をご覧ください。

1. ターミナル ウィンドウを開き、このラボで使用している Lab08 フォルダーに移動します。

1. Azure Functions のコマンドライン サポートをインストールするには、`node.js` が必要です。

1. ターミナルペインで、以下を実行して node のバージョンを確認します。

   ```sh
   node --version
   ```

   > node.jsがインストールされていない場合 [ダウンロードはこちら](https://docs.npmjs.com/getting-started/installing-node#osx-or-windows). 8.5`より古いバージョンを使用している場合は、以下を実行してください：

   ```sh
   npm i -g node@latest
   ```

1. 以下のコマンドを入力して実行し、Azure Functionツールをダウンロードする：

   ```sh
   npm install -g azure-functions-core-tools
   ```

   > このコマンドが失敗した場合は、前のステップを参照してnode.jsをセットアップしてください。これらの変更を有効にするには、ターミナル・ウィンドウを再起動する必要があるかもしれません。

1. ターミナルペインで、以下のコマンドを入力して実行します。このコマンドは、新しい Azure Functions プロジェクトを作成します：

   ```sh
   func init ChangeFeedFunctions
   ```

1. プロンプトが表示されたら、**dotnet** ワーカーランタイムを選択します。矢印キーを使って上下にスクロールする。

1. ディレクトリを前のステップで作成した `ChangeFeedFunctions` ディレクトリに変更する。

   ```sh
   cd ChangeFeedFunctions
   ```

1. ターミナル・ペインで、以下のコマンドを入力して実行する：

   ```sh
   func new
   ```

1. プロンプトが表示されたら、言語リストから**C#**を選択します。矢印キーで上下にスクロールします。

1. プロンプトが表示されたら、テンプレートのリストから**CosmosDBTrigger**を選択します。矢印キーを使用して、上下にスクロールします。

1. プロンプトが表示されたら、関数の名前 `MaterializedViewFunction` を入力します。

1. ChangeFeedFunctions.csproj** ファイルを開き、ターゲットフレームワークを .NET Core 3.1 に更新します。

    ```xml
   <TargetFramework>netcoreapp3.1</TargetFramework>
    ```

1. ターミナル・ペインで以下のコマンドを入力し、実行する：

   ```sh
   dotnet add package Microsoft.Azure.Cosmos
   dotnet add package Microsoft.NET.Sdk.Functions --version 3.0.9
   dotnet add package Microsoft.Azure.WebJobs.Extensions.CosmosDB --version 3.0.7
   dotnet add ChangeFeedFunctions.csproj reference ..\\Shared\\Shared.csproj
   ```

1. ターミナル・ペインでプロジェクトをビルドする：

   ```sh
   dotnet build
   ```

1. Visual Studio Code で新しい **ChangeFeedFunctions** フォルダを開き、**local.settings.json** と **MaterializedViewFunction.cs** ファイルを探します。

## Materialized ViewパターンにCosmos DB Change Feedを使用する

Materialized Viewパターンは、ソースデータの形式がアプリケーションの要件に適していない環境で、事前に入力されたデータのビューを生成するために使用されます。この例では、都道府県別に集計された販売データのリアルタイムコレクションを作成し、別のアプリケーションが販売データのサマリーをすばやく取得できるようにします。

### マテリアライズド・ビュー Azure 関数の作成

1. local.settings.json**ファイルを探し、選択してエディターで開きます。

1. このラボで以前に収集した Cosmos DB アカウントの **Primary Connection String** パラメータを使用して、新しい値 `DBConnection` を追加します。**local.settings.json**ファイルはこのようになっているはずです：

   ```json
   {
     "IsEncrypted": false,
     "Values": {
       "AzureWebJobsStorage": "UseDevelopmentStorage=true",
       "FUNCTIONS_WORKER_RUNTIME": "dotnet",
       "DBConnection": "<your-db-connection-string>"
     }
   }
   ```

1. 新しい`MaterializedViewFunction.cs`ファイルを選択してエディタで開きます。

   > databaseName**、**collectionName**、および**ConnectionStringSetting**は、関数が変更をリッスンするソースCosmos DBアカウントを参照します。

1. databaseName**値を `StoreDatabase` に変更します。

1. collectionName**値を `CartContainerByState` に変更する。

   > Cosmos DB の変更フィードはパーティション内の順序が保証されているので、この場合、パーティションが既に State に設定されているコンテナ、`CartContainerByState` をソースとして使用します。

1. ConnectionStringSetting**プレースホルダを、先ほど追加した新しい設定**DBConnection**に置き換えます。

   ```csharp
   ConnectionStringSetting = "DBConnection",
   ```

1. ConnectionStringSetting**と**LeaseCollectionName**の間に以下の行を追加する：

   ```csharp
   CreateLeaseCollectionIfNotExists = true,
   ```

1. LeaseCollectionName*** の値を `materializedViewLeases` に変更します。

   > リースコレクションはCosmos DB Change Feedの重要な部分です。リースコレクションは、Cosmos DBの変更フィードの重要な部分です。リースコレクションは、関数の複数のインスタンスがコレクションを操作することを可能にし、関数が最後に終了した場所を示す仮想の_bookmark_として機能します。

1. これで、**Run**関数は次のようになります：

   ```csharp
   [FunctionName("MaterializedViewFunction")]
   public static void Run([CosmosDBTrigger(
      databaseName: "StoreDatabase",
      collectionName: "CartContainerByState",
      ConnectionStringSetting = "DBConnection",
      CreateLeaseCollectionIfNotExists = true,
      LeaseCollectionName = "materializedViewLeases")]IReadOnlyList<Document> input, ILogger log)
   {
      if (input != null && input.Count > 0)
      {
         log.LogInformation("Documents modified " + input.Count);
         log.LogInformation("First document Id " + input[0].Id);
      }
   }
   ```

> この関数は、コンテナを一定間隔でポーリングし、前回のリース時からの変更点をチェックすることで動作する。この関数を実行するたびに、複数のドキュメントが変更される可能性があるため、入力は IReadOnlyList of Documents となります。

1. MaterializedViewFunction.cs`ファイルの先頭に以下のusing文を追加します：

   ```csharp
   using System.Threading.Tasks;
   using System.Linq;
   using Newtonsoft.Json;
   using Microsoft.Azure.Cosmos;
   using Shared;
   ```

1. **Run**関数のシグネチャを `Task` 戻り値の型で `async` に変更します。これで関数は以下のようになります：

   ```csharp
   [FunctionName("MaterializedViewFunction")]
      public static async Task Run([CosmosDBTrigger(
         databaseName: "StoreDatabase",
         collectionName: "CartContainerByState",
         ConnectionStringSetting = "DBConnection",
         CreateLeaseCollectionIfNotExists = true,
         LeaseCollectionName = "materializedViewLeases")]IReadOnlyList<Document> input, ILogger log)
      {
         if (input != null && input.Count > 0)
         {
            log.LogInformation("Documents modified " + input.Count);
            log.LogInformation("First document Id " + input[0].Id);
         }
      }
   ```

1. 今回のターゲットは **StateSales** というコンテナです。MaterializedViewFunction**の先頭に以下の行を追加し、接続先の設定を行います。エンドポイントのURLとキーは必ず置き換えてください。

   ```csharp
    private static readonly string _endpointUrl = "<your-endpoint-url>";
    private static readonly string _primaryKey = "<your-primary-key>";
    private static readonly string _databaseId = "StoreDatabase";
    private static readonly string _containerId = "StateSales";
    private static CosmosClient _client = new CosmosClient(_endpointUrl, _primaryKey);
   ```

### StateSalesデータ用の新しいクラスを追加する。

1. エディタで **Shared** フォルダ内の `DataModel.cs` を開きます。

1. CartAction**クラスの定義に続いて、以下のように新しいクラスを追加します：

   ```csharp
   public class StateCount
   {
      [JsonProperty("id")]
      public string Id { get; set; }
      public string State { get; set; }
      public int Count { get; set; }
      public double TotalSales { get; set; }

      public StateCount()
      {
         Id = Guid.NewGuid().ToString();
      }
   }
   ```

### MaterializedViewFunction を更新してマテリアライズド・ビューを作成する

Azure関数は、変更されたドキュメントのリストを受け取ります。このリストを、各ドキュメントの状態をキーにした辞書に整理し、購入したアイテムの合計価格と数を追跡したいと思います。このディクショナリを使用して、後でマテリアライズド・ビュー・コレクション **StateSales** にデータを書き込みます。

1. エディタで **MaterializedViewFunction.cs** ファイルに戻ります。

1. MaterializedViewFunction.cs**のコードで次のセクションを探します。

   ```csharp
   if (input != null && input.Count > 0)
   {
      log.LogInformation("Documents modified " + input.Count);
      log.LogInformation("First document Id " + input[0].Id);
   }
   ```

1. 2つのロギング行を以下のコードに置き換える：

   ```csharp
   var stateDict = new Dictionary<string, List<double>>();
   foreach (var doc in input)
   {
      var action = JsonConvert.DeserializeObject<CartAction>(doc.ToString());

      if (action.Action != ActionType.Purchased)
      {
         continue;
      }

      if (stateDict.ContainsKey(action.BuyerState))
      {
         stateDict[action.BuyerState].Add(action.Price);
      }
      else
      {
         stateDict.Add(action.BuyerState, new List<double> { action.Price });
      }
   }
   ```

1. この_foreach_ループの最後に、以下のコードを追加して、デスティネーション・コンテナに接続する：

   ```csharp
      var database = _client.GetDatabase(_databaseId);
      var container = database.GetContainer(_containerId);

      //todo - Next steps go here
   ```

1. 集約コレクションを扱うので、辞書の各エントリに対してドキュメントを作成または更新することになります。手始めに、気になるドキュメントが存在するかどうかをチェックする必要があります。上記の `todo` 行の後に以下のコードを追加します：

   ```csharp
   var tasks = new List<Task>();

   foreach (var key in stateDict.Keys)
   {
      var query = new QueryDefinition("select * from StateSales s where s.State = @state").WithParameter("@state", key);

      var resultSet = container.GetItemQueryIterator<StateCount>(query, requestOptions: new QueryRequestOptions() { PartitionKey = new Microsoft.Azure.Cosmos.PartitionKey(key), MaxItemCount = 1 });

      while (resultSet.HasMoreResults)
      {
         var stateCount = (await resultSet.ReadNextAsync()).FirstOrDefault();

         if (stateCount == null)
         {
            //todo: Add new doc code here
         }
         else
         {
            //todo: Add existing doc code here
         }

         //todo: Upsert document
      }
   }

   await Task.WhenAll(tasks);
   ```

   > CreateItemQuery**呼び出しの_maxItemCount_に注意してください。各ステートには最大でも1つのドキュメントしかないため、最大でも1つの結果しか期待できません。

1. stateCountオブジェクトが_null_の場合は、新しいオブジェクトを作成します。以下のコードで `//todo: Add new doc code here` セクションを置き換えてください：

   ```csharp
   stateCount = new StateCount();
   stateCount.State = key;
   stateCount.TotalSales = stateDict[key].Sum();
   stateCount.Count = stateDict[key].Count;
   ```

1. stateCountオブジェクトが存在する場合は、それを更新する。以下のコードで `//todo: Add existing doc code here` セクションを置き換える：

   ```csharp
    stateCount.TotalSales += stateDict[key].Sum();
    stateCount.Count += stateDict[key].Count;
   ```

1. 最後に、保存先のCosmos DBアカウントで_upsert_（更新または挿入）操作を行います。Upsert document` セクションを以下のコードに置き換えます：

   ```csharp
   log.LogInformation("Upserting materialized view document");
   tasks.Add(container.UpsertItemAsync(stateCount, new Microsoft.Azure.Cosmos.PartitionKey(stateCount.State)));
   ```

   > ここでタスクのリストを使っているのは、アップサートを並行して行えるからだ。

1. これで、**MaterializedViewFunction** は次のようになります：

   ```csharp
   using System.Collections.Generic;
   using System.Threading.Tasks;
   using Microsoft.Azure.Documents;
   using Microsoft.Azure.WebJobs;
   using Microsoft.Azure.WebJobs.Host;
   using Microsoft.Extensions.Logging;
   using System.Linq;
   using Newtonsoft.Json;
   using Microsoft.Azure.Cosmos;
   using Shared;

   namespace ChangeFeedFunctions
   {
      public static class MaterializedViewFunction
      {
         private static readonly string _endpointUrl = "<your-endpoint-url>";
         private static readonly string _primaryKey = "<primary-key>";
         private static readonly string _databaseId = "StoreDatabase";
         private static readonly string _containerId = "StateSales";
         private static CosmosClient _client = new CosmosClient(_endpointUrl, _primaryKey);

         [FunctionName("MaterializedViewFunction")]
         public static async Task Run([CosmosDBTrigger(
            databaseName: "StoreDatabase",
            collectionName: "CartContainerByState",
            ConnectionStringSetting = "DBConnection",
            CreateLeaseCollectionIfNotExists = true,
            LeaseCollectionName = "materializedViewLeases")]IReadOnlyList<Document> input, ILogger log)
         {
            if (input != null && input.Count > 0)
            {
               var stateDict = new Dictionary<string, List<double>>();

               foreach (var doc in input)
               {
                  var action = JsonConvert.DeserializeObject<CartAction>(doc.ToString());

                  if (action.Action != ActionType.Purchased)
                  {
                     continue;
                  }

                  if (stateDict.ContainsKey(action.BuyerState))
                  {
                     stateDict[action.BuyerState].Add(action.Price);
                  }
                  else
                  {
                     stateDict.Add(action.BuyerState, new List<double> { action.Price });
                  }
               }

               var database = _client.GetDatabase(_databaseId);
               var container = database.GetContainer(_containerId);

               var tasks = new List<Task>();

               foreach (var key in stateDict.Keys)
               {
                  var query = new QueryDefinition("select * from StateSales s where s.State = @state").WithParameter("@state", key);

                  var resultSet = container.GetItemQueryIterator<StateCount>(query, requestOptions: new QueryRequestOptions() { PartitionKey = new Microsoft.Azure.Cosmos.PartitionKey(key), MaxItemCount = 1 });

                  while (resultSet.HasMoreResults)
                  {
                     var stateCount = (await resultSet.ReadNextAsync()).FirstOrDefault();

                     if (stateCount == null)
                     {
                        stateCount = new StateCount();
                        stateCount.State = key;
                        stateCount.TotalSales = stateDict[key].Sum();
                        stateCount.Count = stateDict[key].Count;
                     }
                     else
                     {
                        stateCount.TotalSales += stateDict[key].Sum();
                        stateCount.Count += stateDict[key].Count;
                     }

                     log.LogInformation("Upserting materialized view document");
                     tasks.Add(container.UpsertItemAsync(stateCount, new Microsoft.Azure.Cosmos.PartitionKey(stateCount.State)));
                  }
               }

               await Task.WhenAll(tasks);
            }
         }
      }
   }
   ```

### マテリアライズド・ビュー機能の動作確認テスト

1. ターミナル・ウィンドウを3つ開く。

1. 最初のターミナル・ウィンドウで、**DataGenerator** フォルダーに移動します。

1. 最初の**ターミナル・ウィンドウに以下を入力して実行することにより、**DataGenerator**を起動する：

   ```sh
   dotnet run
   ```

1. 2番目の**ターミナルウィンドウで、**ChangeFeedConsole**フォルダに移動する。

1. 2 番目の** ターミナル・ウィンドウで以下を入力・実行し、**ChangeFeedConsole** コンシューマを起動します：

   ```sh
   dotnet run
   ```

1. 3番目の**ターミナルウィンドウで、**ChangeFeedFunctions**フォルダに移動します。

1. 3番目の**ターミナルウィンドウで、以下を入力して実行し、Azure Functionsを起動します：

   ```sh
   func host start
   ```

   > プロンプトが表示されたら、**アクセスを許可** を選択します。

   > データはDataGenerator > CartContainer > ChangeFeedConsole > CartContainerByState > MaterializedViewFunction > StateSalesから渡されます。

1. 最初の**ウィンドウには、データが生成されていることを示すアスタリスクが表示され、**2番目の**ウィンドウと**3番目の**ウィンドウには、関数が実行されていることを示すコンソールメッセージが表示されるはずです。

1. ブラウザウィンドウを開き、Cosmos DBリソースのData Explorerに移動します。

1. StoreDatabase**を展開し、次に**StateSales**を展開し、**Items**を選択します。

1. コンテナに状態別にデータが入力されているのが見えるはずです。

   CosmosDBのStateSalesコンテナが表示される](../media/08-cosmos-state-sales.jpg "Browse the StateSales container items")

1. 最初のターミナルウィンドウで、いずれかのキーを押してデータ生成を停止する。

1. 2番目の**ターミナルウィンドウで、いずれかのキーを押してデータ移行を停止する。

1. 3番目の**ターミナル・ウィンドウで、コンソールのログ・メッセージが停止するのを待ち、関数がデータ処理を終了するのを待つ。数秒しかかからないはずである。それから `Ctrl+C` を押して関数の実行を終了する。

## Azure Cosmos DB Change Feed を使って Azure Functions を使って EventHub にデータを書き込む

このラボの最後の Change Feed の使用例では、Azure Event Hub に変更データを書き出す簡単な Azure Function を作成します。ストリーム プロセッサを使用して、Power BI で利用できるリアルタイムのデータ出力を作成し、e コマース ダッシュボードを構築します。

### Power BI アカウントの作成（オプション）

ダッシュボードを作成するラボに従わない場合は、この手順を省略できます。

> Power BI アカウントにサインアップするには、[Power BI サイト](https://powerbi.microsoft.com/en-us/) にアクセスし、**無料サインアップ** を選択します。

1. ログインしたら、**CosmosDB** という新しいワークスペースを作成します。

### Azure Event Hub 接続情報の取得

1. Azure Portal](https://portal.azure.com) に切り替える。

1. ポータルの左側で、**Resource groups** リンクを選択します。

   ![リソースグループがハイライトされます](../media/08-select-resource-groups.jpg "Browse to resource groups")

1. リソースグループ**ブレードで、**cosmoslabs**リソースグループを見つけて選択します。

   ![ラボのリソースグループがハイライトされます](../media/08-cosmos-in-resources.jpg "Select your lab resource group")

1. cosmoslabs**リソースブレードで、Event Hub名前空間を選択します。

   ![ラボのイベントハブが強調表示されます](../media/08-cosmos-select-hub.jpg "Select the lab Event Hub resource")

1. Event Hub** ブレードで、**Settings** の下にある **Shared Access Policies** を見つけて選択します。

1. Shared Access Policies** ブレードで、ポリシー **RootManageSharedAccessKey** を選択します。

1. 表示されるパネルで、**Connection string-primary key** の値をコピーし、後でこの実習ラボで使用するために保存します。

   ![イベント ハブのキーがハイライト表示されます](../media/08-event-hub-keys.jpg "Copy and save the connection string for later use")

### Azure Stream Analyticsジョブの出力を作成します。

Power BI に接続して Event Hub を可視化したくない場合は、このステップは省略できます。

1. ブラウザの **cosmoslabs** ブレードに戻ります。

1. cosmoslabs**リソース・ブレードで、ストリーム分析ジョブを選択します。

   ![ストリーム分析がハイライトされます](../media/08-select-stream-processor.jpg "Select the stream analytics resource")

1. CartStreamProcessor**概要画面で**Outputs**を選択します。

   ストリーム分析リソースの概要ブレードが表示されます](../media/08-stream-processor-output.jpg "Review the overview blade")

1. Outputs**ページの上部で、**+Add**を選択し、**Power BI**を選択します。

   ![Power BIがハイライトされます](../media/08-add-power-bi.jpg "Power BIを選択")

1. Authorize**ボタンを選択し、ログインプロンプトに従って Power BI アカウントでこの出力を認証します。

1. 表示されるウィンドウで、以下のデータを入力します。

   - 出力エイリアスを `averagePriceOutput` に設定します。

   - Group workspace_ を `CosmosDB` または Power BI で新しいワークスペースを作成したときに使用した名前に設定します。

   - データセット名を `averagePrice` に設定します。

   - テーブル名を `averagePrice` に設定します。

   - 認証モード］を［ユーザートークン］に設定します。

   - Save**を選択する。

   ![新規出力ダイアログが表示される](../media/08-adding-output.jpg "値を設定して保存を選択")

1. 2つ目の出力を追加するには、前のステップを繰り返します。

   - 出力エイリアスを `incomingRevenueOutput` に設定する。

   - グループワークスペース_ を `cosmosdb` に設定する。

   - データセット名を `incomingRevenue` に設定する。

   - テーブル名を `incomingRevenue` に設定する。

   - 認証モードを `User token` に設定する。

   - 保存** を選択する。

1. 前のステップを繰り返して、3つ目の出力を追加します。

   - 出力エイリアスを`top5Output`に設定する。

   - Set _Group workspace_ to `cosmosdb`.

   - データセット名を `top5` に設定する。

   - テーブル名を `top5` に設定する。

   - 認証モードを `User token` に設定する。

   - 保存** を選択する。

1. 前のステップを繰り返し、4つ目の（そして最後の）出力を追加する。

   - 出力エイリアスを `uniqueVisitorCountOutput` に設定する。

   - グループワークスペース_を`cosmosdb`に設定する。

   - データセット名を `uniqueVisitorCount` に設定する。

   - テーブル名を `uniqueVisitorCount` に設定する。

   - 認証モードを `User token` に設定する。

   - 保存**を選択する

1. これらの手順を完了すると、**Outputs** ブレードは以下のように表示されます：

   ![出力ブレードには4つの出力が表示されます](../media/08-outputs-blade.jpg "You should see four outputs now")

### イベントハブにデータを書き込むAzure Functionの作成

すべての設定が終わったところで、新しいEvent Hubにリアルタイムで変更データを書き込むためのAzure Functionを書くのがいかに簡単かわかるだろう。

1. ターミナルウィンドウを開き、**ChangeFeedFunctions** フォルダに移動する。

1. 以下のコマンドを入力して実行し、新しい関数を作成します：

   ```sh
   func new
   ```

   1. プロンプトが表示されたら、テンプレートとして**CosmosDBTrigger**を選択します。

   1. プロンプトが表示されたら、_name_ に `AnalyticsFunction` を入力します。

1. 以下のように入力して実行し、[Microsoft Azure Event Hubs](https://www.nuget.org/packages/Microsoft.Azure.EventHubs/) NuGet パッケージを追加します：

   ```sh
   dotnet add package Microsoft.Azure.EventHubs --version 4.3.0
   ```

1. 新しい**AnalyticsFunction.cs**ファイルを選択し、エディターで開きます。

   ```csharp
   using Microsoft.Azure.EventHubs;
   using System.Threading.Tasks;
   using System.Text;
   ```

1. 設定で**Run**関数のシグネチャを変更する。

   - **databaseName** を `StoreDatabase`
   - **collectionName** を `CartContainer`
   - **ConnectionStringSetting** を `DBConnection`
   - **LeaseCollectionName** を `analyticsLeases`.

1. ConnectionStringSetting**と**LeaseCollectionName**の間に以下の行を追加する：

   ```csharp
   CreateLeaseCollectionIfNotExists = true,
   ```

1. Run**関数を`async`に変更する。コードファイルは次のようになるはずだ：

   ```csharp
   using System.Collections.Generic;
   using Microsoft.Azure.Documents;
   using Microsoft.Azure.WebJobs;
   using Microsoft.Azure.WebJobs.Host;
   using Microsoft.Extensions.Logging;
   using Microsoft.Azure.EventHubs;
   using System.Threading.Tasks;
   using System.Text;

   namespace ChangeFeedFunctions
   {
       public static class AnalyticsFunction
       {
           [FunctionName("AnalyticsFunction")]
           public static async Task Run([CosmosDBTrigger(
               databaseName: "StoreDatabase",
               collectionName: "CartContainer",
               ConnectionStringSetting = "DBConnection",
               CreateLeaseCollectionIfNotExists = true,
               LeaseCollectionName = "analyticsLeases")]IReadOnlyList<Document> input, ILogger log)
           {
               if (input != null && input.Count > 0)
               {
                   log.LogInformation("Documents modified " + input.Count);
                   log.LogInformation("First document Id " + input[0].Id);
               }
           }
       }
   }
   ```

1. クラスの一番上に、以下のコンフィギュレーション・パラメーターを追加する：

   ```csharp
   private static readonly string _eventHubConnection = "<event-hub-connection>";
   private static readonly string _eventHubName = "carteventhub";
   ```

1. EventHubConnection**のプレースホルダを、以前に収集したイベントハブの**Connection string-primary key**の値で置き換える。

1. ロギングしている2行を以下のコードに置き換えて、**EventHubClient**を作成することから始めます：

   ```csharp
   var sbEventHubConnection = new EventHubsConnectionStringBuilder(_eventHubConnection){
       EntityPath = _eventHubName
   };

   var eventHubClient = EventHubClient.CreateFromConnectionString(sbEventHubConnection.ToString());

   //todo: Next steps here
   ```

1. 変更された各ドキュメントについて、Event Hubにデータを書き出したい。幸い、JSONデータを期待するようにEvent Hubを設定したので、ここで行う処理はほとんどない。以下のコード・スニペットを追加します。

   ```csharp
   var tasks = new List<Task>();

   foreach (var doc in input)
   {
       var json = doc.ToString();

       var eventData = new EventData(Encoding.UTF8.GetBytes(json));

       log.LogInformation("Writing to Event Hub");
       tasks.Add(eventHubClient.SendAsync(eventData));
   }

   await Task.WhenAll(tasks);
   ```

1. AnalyticsFunction**の最終バージョンは以下のようになる：

   ```csharp
   using System.Collections.Generic;
   using Microsoft.Azure.Documents;
   using Microsoft.Azure.WebJobs;
   using Microsoft.Azure.WebJobs.Host;
   using Microsoft.Extensions.Logging;
   using Microsoft.Azure.EventHubs;
   using System.Threading.Tasks;
   using System.Text;

   namespace ChangeFeedFunctions
   {
       public static class AnalyticsFunction
       {
           private static readonly string _eventHubConnection = "<your-connection-string>";
           private static readonly string _eventHubName = "carteventhub";

           [FunctionName("AnalyticsFunction")]
           public static async Task Run([CosmosDBTrigger(
               databaseName: "StoreDatabase",
               collectionName: "CartContainer",
               ConnectionStringSetting = "DBConnection",
               CreateLeaseCollectionIfNotExists = true,
               LeaseCollectionName = "analyticsLeases")]IReadOnlyList<Document> input, ILogger log)
           {
               if (input != null && input.Count > 0)
               {
                   var sbEventHubConnection = new EventHubsConnectionStringBuilder(_eventHubConnection)
                   {
                       EntityPath = _eventHubName
                   };

                   var eventHubClient = EventHubClient.CreateFromConnectionString(sbEventHubConnection.ToString());

                   var tasks = new List<Task>();

                   foreach (var doc in input)
                   {
                       var json = doc.ToString();

                       var eventData = new EventData(Encoding.UTF8.GetBytes(json));

                       log.LogInformation("Writing to Event Hub");
                       tasks.Add(eventHubClient.SendAsync(eventData));
                   }

                   await Task.WhenAll(tasks);
               }
           }
       }
   }
   ```

### AnalyticsFunction をテストするための Power BI ダッシュボードの作成

1. もう一度、3 つのターミナル・ウィンドウを開きます。

1. 最初のターミナル・ウィンドウで、**DataGenerator** フォルダーに移動します。

1. 最初の**ターミナル・ウィンドウで、以下の行を入力して実行することにより、データ生成プロセスを開始する：

   ```sh
   dotnet run
   ```

1. 2番目の**ターミナルウィンドウで、**ChangeFeedFunctions**フォルダに移動します。

1. 2番目の**ターミナルウィンドウで、以下の行を入力して実行し、Azure Functionsを起動します：

   ```sh
   func host start
   ```

1. 3番目の**ターミナルウィンドウで、**ChangeFeedConsole**フォルダに移動します。

1. 3番目の**ターミナル・ウィンドウで、以下の行を入力して実行し、チェンジ・フィード・コンソール・プロセッサーを起動する：

   ```sh
   dotnet run
   ```

### Power BI で Event Hub の出力データを可視化したくない場合は、スキップしてください。

1. 次のステップに進む前に、データ ジェネレーターが実行されていること、および Azure Functions と Console Change Processor が起動していることを確認します。

1. **CartStreamProcessor**の概要画面に戻り、上部の**Start**ボタンを選択してプロセッサーを開始します。プロンプトが表示されたら、出力を開始する **now** を選択します。プロセッサーの起動には数分かかることがあります。

> [!TIP]
> Stream Analytics ジョブの起動に失敗する場合は、イベントハブへの接続に問題がある可能性があります。これを修正するには、Stream Analytics ジョブの **Inputs** に移動して、Service Bus の名前空間とイベントハブ名を確認し、イベントハブへの `cartInput` 接続を削除して再作成してください。

   ![開始リンクがハイライトされている](../media/08-start-processor.jpg "Start the stream analytics job")

   > 続行する前に、プロセッサーの起動を待つ

1. Web ブラウザを開き、**Power BI** Web サイトに移動します。

1. サインインし、左側のセクションから**CosmosDB**を選択します。

   ![The Power BI portal is displayed](../media/08-power-bi.jpg "Open the PowerBI website")

1. 画面右上の **Create** を選択し、**Dashboard** を選択します。

1. ダッシュボード**画面で、上部から**タイルの追加**を選択します。

   ![Add Tile link is highlighted](../media/08-power-bi-add-title.jpg "Add a new tile")

1. カスタムストリーミングデータ**を選択し、**次へ**を押す。

   ![The real-time data streaming tile is highlighted.](../media/08-pbi-custom-streaming-data.jpg "Add a new stream data item")

1. カスタムストリーミングデータタイルの追加**ウィンドウから**平均価格**を選択します。

   ![averagePrice is highlighted](../media/08-add-averageprice-pbi.jpg "Select averagePrice")

1. 可視化タイプ_から**クラスター棒グラフ**を選択します。

   - Axis_の下で**Add Value**を選択し、**Action**を選択します。

   - 値の下で、**値を追加**を選択し、**AVG**を選択します。

   - 次へ**を選択

   ![The settings of the tile are highlighted](../media/08-power-bi-first-tile.jpg "Configure the tile")

   - 平均価格`のような名前をつけて、**適用**を選択します。

1. 以下の同じ手順で、残りの 3 つの入力にタイルを追加する。

   - **incomingRevenue**については、**Axis** を `Time` に設定し、**Values** を `Sum` に設定した**折れ線グラフ** を選択する。表示する時間窓** を少なくとも30分に設定します。

   - **uniqueVisitors**では、**Fields**が `uniqueVisitors`に設定された**Card**を選択してください。

   - **Top5** では、**Axis** を `Item` に設定し、**Value** を `countEvents` に設定した **Clustered column chart** を選択してください。

1. 完了すると、下の画像のようなダッシュボードができ、リアルタイムで更新されます！

   ![最終的なPower BIダッシュボードは、リアルタイムのデータが流れるように表示される。](../media/08-power-bi-dashboard.jpg "Review the new dashboard")

> これが最終ラボの場合は、[ラボ資産の削除](11-cleaning_up.md) の手順に従って、すべてのラボ リソースを削除してください。
