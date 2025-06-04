# Gail_rsi-trade-strategy
Deep-dive analysis project using Python on GAIL INDIA 5 year Data set

Download Dataset from drive it's downloaded from NSE website "https://drive.google.com/drive/folders/1SkBg70Hve3W61jd56tzU8btZ1z7hpuUz?usp=drive_link"

**Install All required libraries**
```python
!pip install pandas numpy matplotlib
```
**Import All required libraries**

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
```
**Read the Excel file and convert it into dataframe.**
```python
df=pd.read_excel('/content/GAIL_2.xlsx')
```
**Remove all extra space from the column because it can be error when you forgot to write extra spaces.**
```python
df.columns = df.columns.str.replace(' ', '', regex=False)
```
**Checks all columns is spaces removed.**
```python
print(df.columns)
```
**Checks all columns and data types to know Dataset**
```python
df.info()
```
**Checks Duplicate data. Remove if any present.**
```python
print(df.duplicated().sum())
```
**Adding one more column of day.**
```python
df['day'] = df['Date'].dt.day_name()
```
**Remove the day which is Saturday or Sunday because Mostly theses days used for special trading.**
```python
df = df[~df['Date'].dt.day_name().isin(['Saturday', 'Sunday'])]
```
**Sorting, Reset index after sorting**
```python
df = df.sort_values(by='Date', ascending=True)
df = df.reset_index(drop=True)
```
**--- RSI Calculation (14-day, using exponential moving average) ---**

```python
delta = df['close'].diff()

gain = delta.clip(lower=0)
loss = -delta.clip(upper=0)

avg_gain = gain.ewm(alpha=1/14, adjust=False).mean()
avg_loss = loss.ewm(alpha=1/14, adjust=False).mean()

rs = avg_gain / avg_loss
df['RSI'] = 100 - (100 / (1 + rs))
```
**Find the RSI value at lower than 30.**
```python
(df['RSI']<30).value_counts()
df[df['RSI'] < 30]
```
**Get signal indices with the 10-day gap rule.(If i get any signal today then no signal for next 10- day)**
```python
signal_indices = []
i = 0
while i < len(df):
    if df.loc[i, 'RSI'] < 30:
        signal_indices.append(i)
        i += 10  # skip 10 days after signal
    else:
        i += 1
```
**Generate 2D profit/loss array from day 3 to 20 and Convert to DataFrame.**
```python
holding_periods = range(3, 21)
pl_matrix = []

for idx in signal_indices:
    row = []
    for hp in holding_periods:
        if idx + hp < len(df):
            buy_price = df.loc[idx, 'close']
            sell_price = df.loc[idx + hp, 'close']
            row.append(sell_price - buy_price)
        else:
            row.append(np.nan)
    pl_matrix.append(row)

pl_df = pd.DataFrame(pl_matrix, columns=[f'Day_{hp}' for hp in holding_periods])
pl_df
```
**Summary of profit/Loss**
```python
summary = []

for col in pl_df.columns:
    values = pl_df[col].dropna()

    avg_return = values.mean()

    profitable = values[values > 0]
    losing = values[values < 0]

    profit_count = len(profitable)
    loss_count = len(losing)
    total = len(values)

    profit_pct = (profit_count / total) * 100 if total > 0 else 0
    loss_pct = (loss_count / total) * 100 if total > 0 else 0

    avg_profit_trade = profitable.mean() if profit_count > 0 else 0
    avg_loss_trade = losing.mean() if loss_count > 0 else 0

    total_profit = profitable.sum() if profit_count > 0 else 0
    total_loss = losing.sum() if loss_count > 0 else 0  # This will be negative

    summary.append({
        'Holding_Day': col,
        'Average_Return': avg_return,
        'Profitable_Trades': profit_count,
        'Losing_Trades': loss_count,
        'Profit_%': profit_pct,
        'Loss_%': loss_pct,
        'Avg_Profit_Trade': avg_profit_trade,
        'Avg_Loss_Trade': avg_loss_trade,
        'Total_Profit': total_profit,
        'Total_Loss': total_loss
    })

# Create summary DataFrame
summary_df = pd.DataFrame(summary)

# sort by Average_Return
summary_df = summary_df.sort_values(by='Average_Return', ascending=False).reset_index(drop=True)

print(summary_df)

```

**Best holding period Calculation**
```python
# Calculate average profit/loss per holding period (column-wise)
avg_returns = pl_df.mean(skipna=True)

# Calculate the best day: maximum average profit
best_day = avg_returns.idxmax()
best_day_value = avg_returns.max()

print(f"✅ Best holding period is: {best_day} with average return of {best_day_value:.2f}")
```
**All trades for Best holding period**
```python
trades_Best = []

for idx in signal_indices:
    exit_idx = idx + 6
    if exit_idx < len(df):
        entry_date = df.loc[idx, 'Date']
        exit_date = df.loc[exit_idx, 'Date']
        entry_price = df.loc[idx, 'close']
        exit_price = df.loc[exit_idx, 'close']
        profit_loss = exit_price - entry_price

        trades_Best.append({
            'Entry_Index': idx,
            'Entry_Date': entry_date,
            'Entry_Price': entry_price,
            'Exit_Date': exit_date,
            'Exit_Price': exit_price,
            'Day_6_PnL': profit_loss
        })

# Convert to DataFrame
trades_Best_df = pd.DataFrame(trades_Best)

print(trades_Best_df)
```



