# Quantitative-Investment
量化交易分析－第四組－台指期交易模組三建置、回測與最佳化參數

## 模組三

### 精神
模組三我們使用python程式最佳化策略的參數，我們總共有7個參數，`M`和`RSI_days`是同步變更:

- M
- a
- N
- RSI_days
- rsi_lower
- rsi_upper
- stop_loss_rate

這些參數分別用於兩種策略：

1. 成交量爆量策略：檢查過去 `N` 天的當日成交量是否是近 `M` 天最大成交量的 `a` 倍。
2. 價格低買高賣策略：根據 RSI (RSI 計算範圍天數 `RSI_days`) 進行交易，當 RSI <= `rsi_lower` 時買入，當 RSI > `rsi_upper` 時賣出，其他情況則不進行交易。
3. 同時，在做空時如果損失超過停損 `stop_loss_rate`，則出清做空部位。

條件一「**成交量爆量**」為必要條件，一定要符合才會做多或做空。因為我們觀察認為成交量突然放大是利空出盡的訊號，買賣方從一致的意見轉為對峙，這時很有機會反轉。

------------

### 回測
我們將原本的 Excel 回測交易策略模型改寫成了 Python 函式 `getStrategyReturn`：

```python
getStrategyReturn(df_original,M,a,N,RSI_days,rsi_lower,rsi_upper,stop_loss_rate)
#省略
#得到策略總報酬
return df.iloc[-1]['對準市值']
```

------------

### 最佳化參數 (Grid Search)
#### 找到最好的參數
接下來，我們使用網格搜索（Grid Search）來進行最佳化參數的選擇。網格搜索是一種通過窮舉所有可能的參數組合來尋找最佳解的方法。

在樣本一中，我們嘗試了 52,920 種參數組合，並根據策略報酬對這些參數組合進行排名。我們找到了最好的參數組合：
- M = 30
- a = 1.1
- N = 5
- RSI_days = 30
- rsi_lower = 35
- rsi_upper = 60
- stop_loss_rate = 0.1
此時的策略報酬為 10,252 元。

#### 但是發現參數不適用於其他樣本
然後我把樣本一52920種參數組合中績效前1000好的參數組合拿出來放入樣本二跟樣本三跑，計算報酬後發現沒有一個參數組合同時在樣本一、樣本二、樣本三超過B&H報酬，所以可以推測可能是over-fitting了，雖然在樣本一中最佳的參數可以跑到10252元報酬，是B&H報酬的3倍左右，當初跑出來的時候直接嚇到下巴掉下來，覺得報酬不可思議，但是依然在樣本二、樣本三跑的不如理想。

#### 我想到解決辦法
為了解決這個問題，後來我加入cross-intersection的策略，這是我自己想出來的策略，我可以在樣本一、二、三都進行相同的52920種參數組合測試績效，然後把各樣本績效前2000好的參數組合取交集，如果有找到10筆，代表那10筆都在樣本一、二、三排名前2000名，所以是好的參數；如果沒有找到交集的參數組合，就繼續尋找前2500名、前3000名、前4000名、前4500名...，一直找下去，相信這個方法可以找到雖然報酬不是最高，但是是在各個樣本中都是前幾高的，更通用、適合各種樣本時段的參數。

#### 解釋程式碼
接下來我要解釋我的程式如何進行網格搜索（Grid Search）最佳化參數:

1. **定義參數空間**：首先，我定義了一個參數空間 `param_grid`，其中包括了所有可能的參數組合。每個參數都有一個範圍或列表，例如，`M_RSI_days` 的範圍是從10到120，步長為10。
```python
# 定義參數範圍
param_grid = {
    'M_RSI_days': list(range(10, 130, 10)), # 10到120之間的整數，步長為10
    'a': [1.0, 1.05, 1.1, 1.15, 1.2], # 1.0到1.2之間的數值，步長為0.05
    'N': list(range(1, 11)), # 1到10的整數
    'rsi_lower': list(range(15, 50, 5)), # 15到45之間的整數，步長為5
    'rsi_upper': list(range(55, 90, 5)), # 55到85之間的整數，步長為5
    'stop_loss_rate': [0.1, 0.15, 0.2, 0.25] # 0.1到0.25之間的數值，步長為0.05
}
```
2. **創建參數網格**：然後，使用 scikit-learn中的 ParameterGrid 模組從參數空間創建一個參數網格，其中包含所有可能的參數組合。
```python
grid = sklearn.model_selection.ParameterGrid(param_grid)
```
3. **初始化最佳回報和最佳參數**：在開始搜索之前，我們初始化最佳回報 (best_return) 為負無窮大，最佳參數 (best_params) 為 None。
```python
best_return = -float("inf")
best_params = None
```
4. 參數搜索：接下來，我們對參數網格中的每一個參數組合進行迭代嘗試（使用 tqdm 來顯示進度條），並計算每組參數的策略回報。如果某組參數的回報優於當前的最佳回報，則更新最佳回報和最佳參數。
```python
results = []
for params in tqdm(grid):
    M_RSI_days = params['M_RSI_days']
    # 計算策略報酬
    StrategyReturn = getStrategyReturn(df, M_RSI_days, params['a'], params['N'], M_RSI_days, params['rsi_lower'], params['rsi_upper'], params['stop_loss_rate'])

    # 如果當前的策略報酬優於之前的最佳報酬，則更新最佳報酬和最佳參數
    if StrategyReturn > best_return:
        best_return = StrategyReturn
        best_params = params

    # 儲存當前的參數和策略報酬
    result = {
        'M': M_RSI_days,
        'a': params['a'],
        'N': params['N'],
        'RSI_days': M_RSI_days,
        'rsi_lower': params['rsi_lower'],
        'rsi_upper': params['rsi_upper'],
        'stop_loss_rate': params['stop_loss_rate'],
        'StrategyReturn': StrategyReturn
    }
    results.append(result)
	```
5. 儲存結果：對於每一組參數，我都會儲存參數值和對應的策略回報。所有的結果都透過results.append(result)儲存在 results list中，然後轉換為 DataFrame。
```
# 將結果列表轉換為 DataFrame
results_df = pd.DataFrame(results)
```
6. 排序和儲存結果：最後，我們按照策略回報（StrategyReturn）進行排序，並將排序後的結果儲存為 CSV 文件。
```python
# StrategyReturn' 欄位進行排序並存檔
results_df = results_df.sort_values(by='StrategyReturn', ascending=False)
results_df.to_csv('strategy_results3.csv', index=False)
```
7. 顯示最佳結果(可以不用，因為前一步驟已經把參數績效存成csv檔案了)：最後，我們顯示前 100 個回報最高的策略及其對應的參數。
```python
top_100_df = results_df.head(100)
print(top_100_df)
```

------------
請參考上述程式碼和說明，以了解模組三的詳細內容和運作原理。
