using System;
using System.Linq;
using cAlgo.API;
using cAlgo.API.Indicators;
using cAlgo.API.Internals;
using cAlgo.Indicators;

namespace cAlgo.Robots
{
    public enum FilterOnOff
    {
        off,
        on
    }
    
    public enum MAType
    {
        SMA,
        EMA,
        HMA
    }
    
    [Robot(TimeZone = TimeZones.UTC, AccessRights = AccessRights.None)]
    public class XAUUSDMACrossVariableImmediate : Robot
    {
        // パラメーター設定
        [Parameter("Short MA Type", DefaultValue = MAType.HMA, Group = "Moving Average")]
        public MAType ShortMAType { get; set; }
        
        [Parameter("Short MA Period", DefaultValue = 9, MinValue = 1, Group = "Moving Average")]
        public int ShortMAPeriod { get; set; }
        
        [Parameter("Long MA Type", DefaultValue = MAType.SMA, Group = "Moving Average")]
        public MAType LongMAType { get; set; }
        
        [Parameter("Long MA Period", DefaultValue = 27, MinValue = 1, Group = "Moving Average")]
        public int LongMAPeriod { get; set; }
        
        // 変動ロット用パラメーター
        [Parameter("Lots Per Million JPY", DefaultValue = 1.0, MinValue = 0.01, Step = 0.01, Group = "Risk Management")]
        public double LotsPerMillion { get; set; }
        
        // 金額ベースの利確設定（万円単位）
        [Parameter("Target Profit (10K JPY)", DefaultValue = 100.0, MinValue = 1.0, Step = 1.0, Group = "Risk Management")]
        public double TargetProfitTenThousand { get; set; }
        
        // テイクプロフィット設定
        [Parameter("Enable Take Profit", DefaultValue = FilterOnOff.off, Group = "Risk Management")]
        public FilterOnOff EnableTakeProfit { get; set; }
        
        [Parameter("Take Profit (pips)", DefaultValue = 200, MinValue = 1, Group = "Risk Management")]
        public double TakeProfitPips { get; set; }
        
        // 陽線/陰線＋価格乖離フィルターパラメーター
        [Parameter("Enable Candle Direction Filter", DefaultValue = FilterOnOff.on, Group = "Candle & Divergence Filter")]
        public FilterOnOff EnableCandleFilter { get; set; }
        
        [Parameter("Min Divergence for Skip (pips)", DefaultValue = 50, MinValue = 1, Group = "Candle & Divergence Filter")]
        public double MinDivergenceForSkip { get; set; }
        
        // インジケーター
        private MovingAverage shortMA;
        private MovingAverage longMA;
        
        // 前回のクロス状態
        private bool wasAbove = false;
        
        // 初回のOnBarをスキップするフラグ
        private bool isFirstBar = true;
        
        // 開始時の残高
        private double startingBalance;
        
        // 目標利益（円）
        private double targetProfitAmount;
        
        protected override void OnStart()
        {
            // 開始時の残高を記録
            startingBalance = Account.Balance;
            
            // 目標利益を円に変換（万円単位から）
            targetProfitAmount = TargetProfitTenThousand * 10000;
            
            // シンボルがXAUUSDでない場合の警告
            if (Symbol.Name != "XAUUSD")
            {
                Print("Warning: This cBot is designed for XAUUSD. Current symbol: " + Symbol.Name);
            }
            
            // Moving Averageの初期化（終値を使用）
            DataSeries priceSource = Bars.ClosePrices;
            
            // Short MAの初期化
            MovingAverageType shortMaType = ConvertToMovingAverageType(ShortMAType);
            shortMA = Indicators.MovingAverage(priceSource, ShortMAPeriod, shortMaType);
            
            // Long MAの初期化
            MovingAverageType longMaType = ConvertToMovingAverageType(LongMAType);
            longMA = Indicators.MovingAverage(priceSource, LongMAPeriod, longMaType);
            
            // 初期状態の設定
            double shortValue = shortMA.Result.Last(1);
            double longValue = longMA.Result.Last(1);
            
            wasAbove = shortValue > longValue;
            
            Print("XAUUSD MA Cross Bot (Variable Lots - Candle Direction Filter) started");
            Print($"Short MA: {ShortMAType} {ShortMAPeriod}");
            Print($"Long MA: {LongMAType} {LongMAPeriod}");
            Print($"Lot Mode: Variable ({LotsPerMillion} lots per 1,000,000 JPY)");
            Print($"Starting Balance: {startingBalance:N0} JPY");
            Print($"Target Profit: {targetProfitAmount:N0} JPY ({TargetProfitTenThousand}万円)");
            
            double targetBalance = startingBalance + targetProfitAmount;
            Print($"Target Balance: {targetBalance:N0} JPY");
            
            // 現在の残高での推定ロット数を表示
            double estimatedLots = CalculateVariableLots();
            Print($"Estimated lots based on current balance: {estimatedLots:F2}");
            
            Print($"Take Profit: {(EnableTakeProfit == FilterOnOff.on ? $"{TakeProfitPips} pips" : "Disabled")}");
            Print($"Candle Direction Filter: {(EnableCandleFilter == FilterOnOff.on ? $"Enabled (Min divergence: {MinDivergenceForSkip} pips)" : "Disabled")}");
            
            // 起動時の既存ポジションを確認
            var existingPositions = Positions.Where(p => p.SymbolName == Symbol.Name).ToList();
            if (existingPositions.Count > 0)
            {
                Print($"Found {existingPositions.Count} existing position(s) for {Symbol.Name}:");
                foreach (var pos in existingPositions)
                {
                    string label = string.IsNullOrEmpty(pos.Label) ? "Manual/Other" : pos.Label;
                    Print($"  - {label} ({pos.TradeType}), Volume: {pos.VolumeInUnits}, P/L: {pos.NetProfit:F2} JPY");
                }
            }
        }
        
        protected override void OnTick()
        {
            // 現在の利益をチェック
            double currentBalance = Account.Balance;
            double currentProfit = currentBalance - startingBalance;
            
            // 目標利益金額に達したかチェック
            if (currentProfit >= targetProfitAmount)
            {
                Print("=== TARGET PROFIT AMOUNT ACHIEVED! ===");
                Print($"Starting Balance: {startingBalance:N0} JPY");
                Print($"Current Balance: {currentBalance:N0} JPY");
                Print($"Profit: {currentProfit:N0} JPY");
                Print($"Target Profit: {targetProfitAmount:N0} JPY ({TargetProfitTenThousand}万円)");
                
                // すべてのポジションをクローズ
                CloseAllPositions();
                
                Print("All positions closed. Stopping cBot...");
                Stop();
            }
        }
        
        protected override void OnBar()
        {
            // 終値ベースでMA値を取得（確定した足）
            double shortMAValue = shortMA.Result.Last(1);
            double longMAValue = longMA.Result.Last(1);
            
            // 現在のクロス状態
            bool isAbove = shortMAValue > longMAValue;
            
            // 初回のOnBarはスキップ（起動時の誤エントリー防止）
            if (isFirstBar)
            {
                isFirstBar = false;
                wasAbove = isAbove;
                Print($"Initial state recorded - Short MA: {shortMAValue:F5}, Long MA: {longMAValue:F5}");
                return;
            }
            
            // 確定足の価格を取得
            double openPrice = Bars.OpenPrices.Last(1);
            double closePrice = Bars.ClosePrices.Last(1);
            bool isBullishCandle = closePrice > openPrice;  // 陽線
            bool isBearishCandle = closePrice < openPrice;  // 陰線
            
            // ゴールデンクロスの検出
            if (!wasAbove && isAbove)
            {
                Print($"Golden Cross detected at {Bars.OpenTimes.Last(1)}");
                Print($"Short MA ({ShortMAType}): {shortMAValue:F5}, Long MA ({LongMAType}): {longMAValue:F5}");
                Print($"Open: {openPrice:F5}, Close: {closePrice:F5}");
                Print($"Candle: {(isBullishCandle ? "Bullish (陽線)" : isBearishCandle ? "Bearish (陰線)" : "Doji")}");
                
                // クロスポイントの推定価格（簡易的にLong MAの値を使用）
                double crossPrice = longMAValue;
                double divergence = Math.Abs(closePrice - crossPrice) / Symbol.PipSize;
                Print($"Price divergence from cross point: {divergence:F1} pips");
                
                bool shouldSkip = false;
                
                if (EnableCandleFilter == FilterOnOff.on)
                {
                    // GCの場合：陽線かつ一定pips以上乖離していたらスキップ
                    if (isBullishCandle && divergence >= MinDivergenceForSkip)
                    {
                        shouldSkip = true;
                        Print($"Entry SKIPPED: Bullish candle with divergence ({divergence:F1} pips) >= threshold ({MinDivergenceForSkip} pips)");
                    }
                    else if (isBullishCandle)
                    {
                        Print($"Bullish candle detected but divergence ({divergence:F1} pips) < threshold ({MinDivergenceForSkip} pips) - Entry allowed");
                    }
                    else
                    {
                        Print($"Not a bullish candle - Entry allowed");
                    }
                }
                
                // ポジションをクローズ（スキップの有無に関わらず）
                CloseAllPositions();
                
                // スキップしない場合のみエントリー
                if (!shouldSkip)
                {
                    HandleGoldenCross();
                }
            }
            // デッドクロスの検出
            else if (wasAbove && !isAbove)
            {
                Print($"Dead Cross detected at {Bars.OpenTimes.Last(1)}");
                Print($"Short MA ({ShortMAType}): {shortMAValue:F5}, Long MA ({LongMAType}): {longMAValue:F5}");
                Print($"Open: {openPrice:F5}, Close: {closePrice:F5}");
                Print($"Candle: {(isBullishCandle ? "Bullish (陽線)" : isBearishCandle ? "Bearish (陰線)" : "Doji")}");
                
                // クロスポイントの推定価格（簡易的にLong MAの値を使用）
                double crossPrice = longMAValue;
                double divergence = Math.Abs(closePrice - crossPrice) / Symbol.PipSize;
                Print($"Price divergence from cross point: {divergence:F1} pips");
                
                bool shouldSkip = false;
                
                if (EnableCandleFilter == FilterOnOff.on)
                {
                    // DCの場合：陰線かつ一定pips以上乖離していたらスキップ
                    if (isBearishCandle && divergence >= MinDivergenceForSkip)
                    {
                        shouldSkip = true;
                        Print($"Entry SKIPPED: Bearish candle with divergence ({divergence:F1} pips) >= threshold ({MinDivergenceForSkip} pips)");
                    }
                    else if (isBearishCandle)
                    {
                        Print($"Bearish candle detected but divergence ({divergence:F1} pips) < threshold ({MinDivergenceForSkip} pips) - Entry allowed");
                    }
                    else
                    {
                        Print($"Not a bearish candle - Entry allowed");
                    }
                }
                
                // ポジションをクローズ（スキップの有無に関わらず）
                CloseAllPositions();
                
                // スキップしない場合のみエントリー
                if (!shouldSkip)
                {
                    HandleDeadCross();
                }
            }
            
            // 状態を更新
            wasAbove = isAbove;
            
            // 現在の利益状況を表示
            double currentBalance = Account.Balance;
            double currentProfit = currentBalance - startingBalance;
            if (Math.Abs(currentProfit) > 100) // 100円以上の変化がある場合のみ表示
            {
                double profitRate = (currentProfit / startingBalance) * 100;
                Print($"Current Profit: {currentProfit:N0} JPY ({profitRate:F2}%) - Target: {targetProfitAmount:N0} JPY");
            }
        }
        
        private void HandleGoldenCross()
        {
            // 即座にロングエントリー
            double volume = CalculateVolume();
            
            // テイクプロフィットの計算（ストップロスは設定しない）
            double? takeProfitPips = EnableTakeProfit == FilterOnOff.on ? TakeProfitPips : (double?)null;
            
            var result = ExecuteMarketOrder(TradeType.Buy, Symbol.Name, volume, "GC Long", null, takeProfitPips);
            
            if (result.IsSuccessful)
            {
                Print($"Long position opened successfully");
                Print($"  - No Stop Loss");
                if (EnableTakeProfit == FilterOnOff.on)
                    Print($"  - Take Profit: {result.Position.TakeProfit:F5} ({TakeProfitPips} pips)");
                Print($"  - Target: {targetProfitAmount:N0} JPY profit");
            }
            else
            {
                Print($"Failed to open long position: {result.Error}");
            }
        }
        
        private void HandleDeadCross()
        {
            // 即座にショートエントリー
            double volume = CalculateVolume();
            
            // テイクプロフィットの計算（ストップロスは設定しない）
            double? takeProfitPips = EnableTakeProfit == FilterOnOff.on ? TakeProfitPips : (double?)null;
            
            var result = ExecuteMarketOrder(TradeType.Sell, Symbol.Name, volume, "DC Short", null, takeProfitPips);
            
            if (result.IsSuccessful)
            {
                Print($"Short position opened successfully");
                Print($"  - No Stop Loss");
                if (EnableTakeProfit == FilterOnOff.on)
                    Print($"  - Take Profit: {result.Position.TakeProfit:F5} ({TakeProfitPips} pips)");
                Print($"  - Target: {targetProfitAmount:N0} JPY profit");
            }
            else
            {
                Print($"Failed to open short position: {result.Error}");
            }
        }
        
        private void CloseAllPositions()
        {
            // XAUUSDのすべてのポジションを取得（cBot起動前のポジションも含む）
            var positions = Positions.Where(p => p.SymbolName == Symbol.Name).ToList();
            
            if (positions.Count > 0)
            {
                Print($"Closing {positions.Count} position(s) for {Symbol.Name}");
                
                foreach (var position in positions)
                {
                    var result = ClosePosition(position);
                    if (result.IsSuccessful)
                    {
                        string label = string.IsNullOrEmpty(position.Label) ? "Manual/Other" : position.Label;
                        Print($"Closed position: {label} ({position.TradeType}) with P/L: {position.NetProfit:F2} JPY");
                    }
                    else
                    {
                        Print($"Failed to close position: {result.Error}");
                    }
                }
            }
        }
        
        private double CalculateVariableLots()
        {
            // 口座残高を100万円単位で計算
            double balance = Account.Balance;
            double millionUnits = balance / 1000000.0;
            
            // 100万円あたりのロット数を掛ける
            double lots = millionUnits * LotsPerMillion;
            
            // 小数点第2位に丸める
            lots = Math.Round(lots, 2);
            
            return lots;
        }
        
        private double CalculateVolume()
        {
            // 変動ロット数を計算
            double lots = CalculateVariableLots();
            Print($"Current balance: {Account.Balance:N0} JPY");
            Print($"Calculated variable lots: {lots} ({LotsPerMillion} lots per 1,000,000 JPY)");
            
            // cTraderのボリューム単位に変換（1ロット = 100,000単位）
            double volume = Symbol.QuantityToVolumeInUnits(lots);
            
            // 最小/最大ボリュームの制限
            volume = Math.Max(Symbol.VolumeInUnitsMin, volume);
            volume = Math.Min(Symbol.VolumeInUnitsMax, volume);
            
            // ボリュームステップに合わせる
            volume = Symbol.NormalizeVolumeInUnits(volume);
            
            Print($"Volume: {volume} units ({lots} lots)");
            
            return volume;
        }
        
        private MovingAverageType ConvertToMovingAverageType(MAType maType)
        {
            switch (maType)
            {
                case MAType.SMA:
                    return MovingAverageType.Simple;
                case MAType.EMA:
                    return MovingAverageType.Exponential;
                case MAType.HMA:
                    return MovingAverageType.Hull;
                default:
                    return MovingAverageType.Simple;
            }
        }
        
        protected override void OnStop()
        {
            Print("XAUUSD MA Cross Bot (Variable Lots - Candle Direction Filter) stopped");
            
            // 最終的な利益状況を表示
            double finalBalance = Account.Balance;
            double totalProfit = finalBalance - startingBalance;
            double finalProfitRate = (totalProfit / startingBalance) * 100;
            
            Print($"=== FINAL RESULTS ===");
            Print($"Starting Balance: {startingBalance:N0} JPY");
            Print($"Final Balance: {finalBalance:N0} JPY");
            Print($"Total Profit: {totalProfit:N0} JPY");
            Print($"Profit Rate: {finalProfitRate:F2}%");
            Print($"Target Profit: {targetProfitAmount:N0} JPY ({TargetProfitTenThousand}万円)");
            
            // Bot停止時の現在のポジション情報を表示
            var position = Positions.FirstOrDefault(p => p.SymbolName == Symbol.Name && 
                                                        (p.Label == "GC Long" || p.Label == "DC Short"));
            if (position != null)
            {
                Print($"Open position: {position.Label}, P/L: {position.NetProfit:F2} JPY");
            }
        }
    }
}
