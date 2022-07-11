---
title: Non-Transactional DML Statements
summary: Learn the non-transactional DML statements in TiDB. At the expense of atomicity and isolation, a DML statement is split into multiple statements to be executed in sequence, which improves the stability and ease of use in batch data processing scenarios.
---

# 非トランザクションDMLステートメント {#non-transactional-dml-statements}

このドキュメントでは、TiDBでの非トランザクションDMLステートメントの使用シナリオ、使用方法、および制限について説明します。さらに、実装の原則と一般的な問題についても説明します。

非トランザクションDMLステートメントは、複数のSQLステートメント（つまり、複数のバッチ）に分割されて順番に実行されるDMLステートメントです。トランザクションのアトミック性と分離を犠牲にして、バッチデータ処理のパフォーマンスと使いやすさを向上させます。

非トランザクションDMLステートメントには`INSERT` 、および`UPDATE`が含まれ、そのうちTiDBは現在`DELETE`のみをサポートしてい`DELETE` 。詳細な構文については、 [`BATCH`](/sql-statements/sql-statement-batch.md)を参照してください。

> **ノート：**
>
> 非トランザクションDMLステートメントは、ステートメントの原子性と分離を保証するものではなく、元のDMLステートメントと同等ではありません。

## 使用シナリオ {#usage-scenarios}

大規模なデータ処理のシナリオでは、多くの場合、大量のデータに対して同じ操作を実行する必要があります。単一のSQLステートメントを使用して操作を直接実行すると、トランザクションサイズが制限を超え、実行パフォーマンスに影響を与える可能性があります。

バッチデータ処理では、多くの場合、時間やデータがオンラインアプリケーションの操作と重複することはありません。並行操作が存在しない場合、分離（ACIDのI）は不要です。バルクデータ操作がべき等であるか、簡単に再試行できる場合も、アトミシティは不要です。アプリケーションがデータ分離も原子性も必要としない場合は、非トランザクションDMLステートメントの使用を検討できます。

非トランザクションDMLステートメントは、特定のシナリオで大規模なトランザクションのサイズ制限をバイパスするために使用されます。 1つのステートメントは、トランザクションを手動で分割する必要があるタスクを完了するために使用され、実行効率が高く、リソース消費が少なくなります。

たとえば、期限切れのデータを削除するには、アプリケーションが期限切れのデータにアクセスしないようにする場合、非トランザクションDMLステートメントを使用して`DELETE`のパフォーマンスを向上させることができます。

## 前提条件 {#prerequisites}

非トランザクションDMLステートメントを使用する前に、以下の条件が満たされていることを確認してください。

-   ステートメントはアトミック性を必要としません。これにより、実行結果で一部の行を変更し、一部の行を変更しないままにすることができます。
-   ステートメントがべき等であるか、エラーメッセージに従ってデータの一部を再試行する準備ができています。システム変数が`tidb_redact_log = 1`と`tidb_nontransactional_ignore_error = 1`に設定されている場合、このステートメントはべき等である必要があります。そうしないと、ステートメントが部分的に失敗したときに、失敗した部分を正確に特定できません。
-   操作対象のデータには他の同時書き込みはありません。つまり、他のステートメントによって同時に更新されることはありません。そうしないと、削除の欠落や誤った削除などの予期しない結果が発生する可能性があります。
-   ステートメントは、ステートメント自体によって読み取られるデータを変更しません。そうしないと、次のバッチが前のバッチによって書き込まれたデータを読み取り、予期しない結果を簡単に引き起こします。
-   ステートメントは[制限](#restrictions)を満たしています。
-   このDMLステートメントによって読み書きされるテーブルに対して同時DDL操作を実行することはお勧めしません。

> **警告：**
>
> `tidb_redact_log`と`tidb_nontransactional_ignore_error`を同時に有効にすると、各バッチの完全なエラー情報が得られない可能性があり、失敗したバッチのみを再試行することはできません。したがって、両方のシステム変数がオンになっている場合、非トランザクションDMLステートメントはべき等である必要があります。

## 使用例 {#usage-examples}

### 非トランザクションDMLステートメントを使用する {#use-a-non-transactional-dml-statement}

次のセクションでは、非トランザクションDMLステートメントの使用について例を挙げて説明します。

次のスキーマを使用してテーブル`t`を作成します。

{{< copyable "" >}}

```sql
CREATE TABLE t (id INT, v INT, KEY(id));
```

```sql
Query OK, 0 rows affected
```

表`t`にいくつかのデータを挿入します。

{{< copyable "" >}}

```sql
INSERT INTO t VALUES (1, 2), (2, 3), (3, 4), (4, 5), (5, 6);
```

```sql
Query OK, 5 rows affected
```

次の操作では、非トランザクションDMLステートメントを使用して、表`t`の列`v`の整数6未満の値の行を削除します。このステートメントは、バッチサイズが2の2つのSQLステートメントに分割され、 `id`列で除算されて実行されます。

{{< copyable "" >}}

```sql
BATCH ON id LIMIT 2 DELETE FROM t WHERE v < 6;
```

```sql
+----------------+---------------+
| number of jobs | job status    |
+----------------+---------------+
| 2              | all succeeded |
+----------------+---------------+
1 row in set
```

上記の非トランザクションDMLステートメントの削除結果を確認してください。

{{< copyable "" >}}

```sql
SELECT * FROM t;
```

```sql
+----+---+
| id | v |
+----+---+
| 5  | 6 |
+----+---+
1 row in set
```

### 実行の進捗状況を確認する {#check-the-execution-progress}

非トランザクションDMLステートメントの実行中に、 `SHOW PROCESSLIST`を使用して進行状況を表示できます。返された結果の`Time`フィールドは、現在のバッチ実行の消費時間を示します。ログと低速ログは、非トランザクションDML実行中の各分割ステートメントの進行状況も記録します。例えば：

{{< copyable "" >}}

```sql
SHOW PROCESSLIST;
```

```sql
+------+------+--------------------+--------+---------+------+------------+----------------------------------------------------------------------------------------------------+
| Id   | User | Host               | db     | Command | Time | State      | Info                                                                                               |
+------+------+--------------------+--------+---------+------+------------+----------------------------------------------------------------------------------------------------+
| 1203 | root | 100.64.10.62:52711 | test   | Query   | 0    | autocommit | /* job 506/500000 */ DELETE FROM `test`.`t1` WHERE `test`.`t1`.`_tidb_rowid` BETWEEN 2271 AND 2273 |
| 1209 | root | 100.64.10.62:52735 | <null> | Query   | 0    | autocommit | show full processlist                                                                              |
+------+------+--------------------+--------+---------+------+------------+----------------------------------------------------------------------------------------------------+
```

### 非トランザクションDMLステートメントを終了します {#terminate-a-non-transactional-dml-statement}

非トランザクションDMLステートメントを終了するには、 `KILL TIDB`を使用できます。次に、TiDBは、現在実行されているバッチの後にすべてのバッチをキャンセルします。ログから実行結果を取得できます。

### バッチ分割ステートメントを照会します {#query-the-batch-dividing-statement}

非トランザクションDMLステートメントの実行中、ステートメントは、DMLステートメントを複数のバッチに分割するために内部的に使用されます。このバッチ分割ステートメントを照会するには、この非トランザクションDMLステートメントに`DRY RUN QUERY`を追加します。その場合、TiDBはこのクエリと後続のDML操作を実行しません。

次のステートメントは、 `BATCH ON id LIMIT 2 DELETE FROM t WHERE v < 6`の実行中にバッチ分割ステートメントを照会します。

{{< copyable "" >}}

```sql
BATCH ON id LIMIT 2 DRY RUN QUERY DELETE FROM t WHERE v < 6;
```

```sql
+--------------------------------------------------------------------------------+
| query statement                                                                |
+--------------------------------------------------------------------------------+
| SELECT `id` FROM `test`.`t` WHERE (`v` < 6) ORDER BY IF(ISNULL(`id`),0,1),`id` |
+--------------------------------------------------------------------------------+
1 row in set
```

### 最初と最後のバッチに対応するステートメントを照会します {#query-the-statements-corresponding-to-the-first-and-the-last-batches}

非トランザクションDMLステートメントの最初と最後のバッチに対応する実際のDMLステートメントを照会するには、この非トランザクションDMLステートメントに`DRY RUN`を追加します。次に、TiDBはバッチを分割するだけで、これらのSQLステートメントを実行しません。バッチが多い場合があるため、すべてのバッチが表示されるわけではなく、最初のバッチと最後のバッチのみが表示されます。

{{< copyable "" >}}

```sql
BATCH ON id LIMIT 2 DRY RUN DELETE FROM t WHERE v < 6;
```

```sql
+-------------------------------------------------------------------+
| split statement examples                                          |
+-------------------------------------------------------------------+
| DELETE FROM `test`.`t` WHERE (`id` BETWEEN 1 AND 2 AND (`v` < 6)) |
| DELETE FROM `test`.`t` WHERE (`id` BETWEEN 3 AND 4 AND (`v` < 6)) |
+-------------------------------------------------------------------+
2 rows in set
```

### オプティマイザヒントを使用する {#use-the-optimizer-hint}

オプティマイザヒントが元々 `DELETE`ステートメントでサポートされている場合、オプティマイザヒントは非トランザクション`DELETE`ステートメントでもサポートされます。ヒントの位置は、通常の`DELETE`ステートメントの位置と同じです。

{{< copyable "" >}}

```sql
BATCH ON id LIMIT 2 DELETE /*+ USE_INDEX(t)*/ FROM t WHERE v < 6;
```

## ベストプラクティス {#best-practices}

非トランザクションDMLステートメントを使用するには、次の手順をお勧めします。

1.  適切な[分割列](#parameter-description)を選択します。整数型または文字列型をお勧めします。
2.  （オプション）非トランザクションDMLステートメントに`DRY RUN QUERY`を追加し、クエリを手動で実行して、DMLステートメントの影響を受けるデータ範囲がおおよそ正しいかどうかを確認します。
3.  （オプション）非トランザクションDMLステートメントに`DRY RUN`を追加し、クエリを手動で実行し、分割ステートメントと実行プランを確認します。インデックス選択の効率に注意を払う必要があります。
4.  非トランザクションDMLステートメントを実行します。
5.  エラーが報告された場合は、エラーメッセージまたはログから特定の失敗したデータ範囲を取得し、手動で再試行または処理してください。

## パラメータの説明 {#parameter-description}

| パラメータ  | 説明                                                                                                                                                                  | デフォルト値                   | 必須かどうか | 推奨値                                             |
| :----- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------ | :----------------------- | :----- | :---------------------------------------------- |
| 分割列    | 上記の非トランザクションDMLステートメント`BATCH ON id LIMIT 2 DELETE FROM t WHERE v < 6`の`id`列など、バッチを分割するために使用される列。                                                                    | TiDBは、分割列を自動的に選択しようとします。 | いいえ    | 最も効率的な方法で`WHERE`の条件を満たすことができる列を選択します。           |
| バッチサイズ | 各バッチのサイズを制御するために使用されます。バッチの数は、DML操作が分割されるSQLステートメントの数です（上記の非トランザクションDMLステートメント`BATCH ON id LIMIT 2 DELETE FROM t WHERE v < 6`の`LIMIT 2`など）。バッチが多いほど、バッチサイズは小さくなります。 | 該当なし                     | はい     | 1000-1000000。バッチが小さすぎたり大きすぎたりすると、パフォーマンスが低下します。 |

### 分割列の選び方 {#how-to-select-a-dividing-column}

非トランザクションDMLステートメントは、データバッチ処理の基礎として列を使用します。これは分割列です。実行効率を高めるには、インデックスを使用するための分割列が必要です。異なるインデックスと分割列によってもたらされる実行効率は、数十回異なる場合があります。分割列を選択するときは、次の提案を考慮してください。

-   アプリケーションデータの分布がわかっている場合は、 `WHERE`の条件に従って、バッチ処理後にデータをより狭い範囲で分割する列を選択します。
    -   理想的には、 `WHERE`条件は分割列のインデックスを利用して、バッチごとにスキャンされるデータの量を減らすことができます。たとえば、各トランザクションの開始時刻と終了時刻を記録するトランザクションテーブルがあり、終了時刻が1か月より前のすべてのトランザクションレコードを削除するとします。トランザクションの開始時刻にインデックスがあり、トランザクションの開始時刻と終了時刻が比較的近い場合は、開始時刻の列を分割列として選択できます。
    -   理想的とは言えないケースでは、分割列のデータ分布は`WHERE`条件から完全に独立しており、分割列のインデックスを使用してデータスキャンの範囲を縮小することはできません。
-   クラスタ化インデックスが存在する場合は、実行効率を高めるために、主キー（主キー`INT`つと主キー`_tidb_rowid`を含む）を分割列として使用することをお勧めします。
-   重複する値が少ない列を選択してください。

分割列を指定しないように選択することもできます。次に、TiDBはデフォルトで`handle`の最初の列を分割列として使用します。ただし、クラスター化インデックスの主キーの最初の列が非トランザクションDMLステートメント（ `ENUM` ）でサポートされて`BIT`ないデータ型である場合、 `SET`はエラーを報告し`JSON` 。アプリケーションのニーズに応じて、適切な分割列を選択できます。

### バッチサイズの設定方法 {#how-to-set-batch-size}

非トランザクションDMLステートメントでは、バッチサイズが大きいほど、分割されるSQLステートメントが少なくなり、各SQLステートメントの実行が遅くなります。最適なバッチサイズは、ワークロードによって異なります。 50000から開始することをお勧めします。バッチサイズが小さすぎるか大きすぎると、実行効率が低下します。

各バッチの情報はメモリに保存されるため、バッチが多すぎるとメモリ消費量が大幅に増加する可能性があります。これは、バッチサイズが小さすぎない理由を説明しています。バッチ情報を格納するための非トランザクションステートメントによって消費されるメモリの上限は[`tidb_mem_quota_query`](/system-variables.md#tidb_mem_quota_query)と同じであり、この制限を超えたときにトリガーされるアクションは、構成項目[`tidb_mem_oom_action`](/system-variables.md#tidb_mem_oom_action-new-in-v610)によって決定されます。

## 制限 {#restrictions}

以下は、非トランザクションDMLステートメントに対する厳しい制限です。これらの制限が満たされていない場合、TiDBはエラーを報告します。

-   1つのテーブルのみを操作できます。マルチテーブル結合は現在サポートされていません。
-   DMLステートメントに`ORDER BY`つまたは`LIMIT`の句を含めることはできません。
-   分割列にはインデックスを付ける必要があります。インデックスは、単一列のインデックス、または結合インデックスの最初の列にすることができます。
-   [`autocommit`](/system-variables.md#autocommit)モードで使用する必要があります。
-   batch-dmlが有効になっている場合は使用できません。
-   [ `tidb_snapshot` ]（/ read-historical-data.md＃operation flow）が設定されている場合は使用できません。
-   `prepare`ステートメントでは使用できません。
-   `ENUM` `BIT`は、 `SET` `JSON`としてサポートされていません。
-   [一時テーブル](/temporary-tables.md)ではサポートされていません。
-   [共通テーブル式](/develop/dev-guide-use-common-table-expression.md)はサポートされていません。

## バッチ実行の失敗を制御する {#control-batch-execution-failure}

非トランザクションDMLステートメントはアトミック性を満たしていません。一部のバッチは成功する可能性があり、一部は失敗する可能性があります。システム変数[`tidb_nontransactional_ignore_error`](/system-variables.md#tidb_nontransactional_ignore_error-new-in-v610)は、非トランザクションDMLステートメントがエラーを処理する方法を制御します。

例外は、最初のバッチが失敗した場合、ステートメント自体が間違っている可能性が高いことです。この場合、非トランザクションステートメント全体が直接エラーを返します。

## 使い方 {#how-it-works}

非トランザクションDMLステートメントの動作原理は、SQLステートメントの自動分割をTiDBに組み込むことです。非トランザクションDMLステートメントがない場合は、SQLステートメントを手動で分割する必要があります。非トランザクションDMLステートメントの動作を理解するには、次のタスクを実行するユーザースクリプトと考えてください。

非トランザクション`BATCH ON $C$ LIMIT $N$ DELETE FROM ... WHERE $P$`の場合、$ C $は分割に使用される列、$ N $はバッチサイズ、$P$はフィルター条件です。

1.  元のステートメントのフィルター条件$P$と、分割用に指定された列$ C $に従って、TiDBは$P$を満たすすべての$C$を照会します。 TiDBは、これらの$C$を$N$に従ってグループ$B_1\ dotsB_k$に分類します。すべての$B_i$について、TiDBは最初と最後の$C$を$S_i$と$E_i$として保持します。このステップで実行されるクエリステートメントは、 [`DRY RUN QUERY`](/non-transactional-dml.md#query-the-batch-dividing-statement)から表示できます。
2.  $ B_i $に含まれるデータは、$ P_i $を満たすサブセットです：$ C $ BETWEEN $ S_i $ AND $E_i$。 $ P_i $を使用して、各バッチが処理する必要のあるデータの範囲を絞り込むことができます。
3.  $ B_i $の場合、TiDBは上記の条件を元のステートメントの`WHERE`条件に埋め込みます。これにより、WHERE（$ P_i $）AND（$ P $）になります。このステップの実行結果は、 [`DRY RUN`](/non-transactional-dml.md#query-the-statements-corresponding-to-the-first-and-the-last-batches)を介して表示できます。
4.  すべてのバッチについて、新しいステートメントを順番に実行します。各グループ化のエラーは収集および結合され、すべてのグループ化が完了した後、非トランザクションDMLステートメント全体の結果として返されます。

## batch-dmlとの比較 {#comparison-with-batch-dml}

batch-dmlは、DMLステートメントの実行中にトランザクションを複数のトランザクションコミットに分割するためのメカニズムです。

> **ノート：**
>
> batch-dmlの使用はお勧めしません。 batch-dml機能が適切に使用されていない場合、データインデックスの不整合のリスクがあります。 batch-dmlは、TiDBの今後のリリースで非推奨になります。

非トランザクションDMLステートメントは、まだすべてのバッチdml使用シナリオに置き換わるものではありません。それらの主な違いは次のとおりです。

-   パフォーマンス： [分割列](#how-to-select-a-dividing-column)が効率的である場合、非トランザクションDMLステートメントのパフォーマンスはbatch-dmlのパフォーマンスに近くなります。分割列の効率が低い場合、非トランザクションDMLステートメントのパフォーマンスはbatch-dmlのパフォーマンスよりも大幅に低くなります。

-   安定性：batch-dmlは、不適切な使用によりデータインデックスの不整合が発生する傾向があります。非トランザクションDMLステートメントは、データインデックスの不整合を引き起こしません。ただし、不適切に使用された場合、非トランザクションDMLステートメントは元のステートメントと同等ではなく、アプリケーションは予期しない動作を観察する可能性があります。詳細については、 [一般的な問題のセクション](#non-transactional-delete-has-exceptional-behavior-that-is-not-equivalent-to-ordinary-delete)を参照してください。

## 一般的な問題 {#common-issues}

### 実際のバッチサイズは、指定されたバッチサイズと同じではありません {#the-actual-batch-size-is-not-the-same-as-the-specified-batch-size}

非トランザクションDMLステートメントの実行中に、最後のバッチで処理されるデータのサイズが、指定されたバッチサイズよりも小さい場合があります。

**重複する値が分割列に存在する**場合、各バッチには、このバッチの分割列の最後の要素の重複する値がすべて含まれます。したがって、このバッチの行数は、指定されたバッチサイズよりも大きい可能性があります。

また、他の同時書き込みが発生した場合、各バッチで処理される行数が指定されたバッチサイズと異なる場合があります。

### <code>Failed to restore the delete statement, probably because of unsupported type of the shard column</code> {#the-code-failed-to-restore-the-delete-statement-probably-because-of-unsupported-type-of-the-shard-column-code-error-occurs-during-execution}

分割列は`ENUM` `BIT`を`JSON`して`SET`ません。新しい分割列を指定してみてください。整数型または文字列型の列を使用することをお勧めします。

選択した分割列がこれらのサポートされていないタイプのいずれでもないときにエラーが発生した場合は、PingCAPテクニカルサポートに連絡してください。

### 非トランザクション<code>DELETE</code>には、通常の<code>DELETE</code>と同等ではない「例外的な」動作があります {#non-transactional-code-delete-code-has-exceptional-behavior-that-is-not-equivalent-to-ordinary-code-delete-code}

非トランザクションDMLステートメントは、このDMLステートメントの元の形式と同等ではありません。これには、次の理由が考えられます。

-   他にも同時書き込みがあります。
-   非トランザクションDMLステートメントは、ステートメント自体が読み取る値を変更します。
-   各バッチで実行されるSQLステートメントは、 `WHERE`の条件が変更されるため、異なる実行プランと式の計算順序を引き起こす可能性があります。したがって、実行結果は元のステートメントとは異なる場合があります。
-   DMLステートメントには、非決定論的な操作が含まれています。

## MySQLの互換性 {#mysql-compatibility}

非トランザクションステートメントはTiDB固有であり、MySQLと互換性がありません。

## も参照してください {#see-also}

-   [`BATCH`](/sql-statements/sql-statement-batch.md)構文
-   [`tidb_nontransactional_ignore_error`](/system-variables.md#tidb_nontransactional_ignore_error-new-in-v610)