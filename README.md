import requests
#from bs4 import BeautifulSoup
import json
import csv
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import urllib.parse
import math
import os
import matplotlib.font_manager as fm
import time






# 1. 초기 Input Data 입력

InvestmentHorizon = 100
Target_Beta = 0.6

BaseBalance = 0
LiquidatedBalance = 0
StockPosition = 0
BondPosition = 0
CashPosition = 100000000

Trading_Log = {}
Operating_Log = []
start = time.time()  # 시작 시간 저장

for i in range(InvestmentHorizon, 0, -1):
    
    Ranking_List = []
    
    data_1 = pd.read_csv('stock_A183700_KBSTAR 채권혼합.csv', index_col=0, encoding='utf-8')
    BM = pd.read_csv('stock_BM_KOSPI.csv', index_col=0, encoding='utf-8')
    
    for code, name in New_CodeList.values:
        data = pd.read_csv(f'stock_A{code}_{name}.csv', index_col=0, encoding='utf-8')

        Add_MA(data, 3)
        Add_MA(data, 20)
        Delta_MA_Percent(data, 3, 20)
        data = data.apply(pd.to_numeric, errors = 'coerce').fillna(0)

        VaR = Calculate_VaR(data, 245, 99)
        Coefficient = round(0.02 / (-VaR / 100), 2)
        Beta = Calculate_Beta(data, BM, 245)
        Daily_Delta_MA = round(data['Delta_MovingAverage'][i], 2)

        Ranking_List.append([code, name, Daily_Delta_MA, VaR, Coefficient, Beta])    


    Ranking_List = pd.DataFrame(Ranking_List)
    Ranking_List = Ranking_List.rename(columns = {0:'Code',1:'Company',2: 'Delta_MA_Percent', 3:'Var', 4:'Coefficient', 5:'Beta'})
    Ranking_List = Ranking_List.sort_values('Delta_MA_Percent', ascending = False)
    Sorted_List = Ranking_List[:5]

    if Sorted_List['Coefficient'].sum() > 1: 
        Sorted_List['weight'] = round(Sorted_List['Coefficient'] / Sorted_List['Coefficient'].sum(), 2)
        Sorted_List['weighted_beta'] = round(Sorted_List['weight'] * Sorted_List['Beta'], 2)
    else: 
        Sorted_List['weight'] = round(Sorted_List['Coefficient'], 2)
        Sorted_List['weighted_beta'] = round(Sorted_List['weight'] * Sorted_List['Beta'], 2)
    
    Sorted_List = pd.DataFrame(Sorted_List.reset_index())
    
    Portfolio_Beta = Sorted_List['weighted_beta'].sum().round(2)
    Bond_Beta = Calculate_Beta(data_1, BM, 245)
    
    Bond_Weight = round((Target_Beta - Portfolio_Beta) / (Bond_Beta - Portfolio_Beta), 2)
    Stock_Weight = round(1-Bond_Weight, 2)
    
    if i == InvestmentHorizon: 
        BaseBalance = CashPosition
    else:
        LiquidatedStockPosition = Liquidate_StockPosition (Trading_Log[BM.index[i+1]], date_index+1)
        LiquidatedBondPosition = Liquidate_BondPosition (Trading_Log[BM.index[i+1]], date_index+1)
        LiquidatedBalance = LiquidatedStockPosition + LiquidatedBondPosition
        BaseBalance = CashPosition + LiquidatedBalance

    
    Trading_format = {'Code':Sorted_List['Code'], 'Title':Sorted_List['Company'], 'Weight': round(Sorted_List['weight']*Stock_Weight, 2), 'Position':round(Sorted_List['weight']*Stock_Weight*BaseBalance, 2)}
    Trading_format = pd.DataFrame(Trading_format)
    Trading_format.loc[5] = {'Code':'183700', 'Title':'KBSTAR 채권혼합', 'Weight': round(Bond_Weight, 2), 'Position':round(Bond_Weight*BaseBalance, 2)}
    Trading_Log[BM.index[i]] = Trading_format
   
       
    StockPosition = Calculate_StockPosition (Trading_format, date_index = i)
    BondPosition = Calculate_BondPosition (Trading_format, date_index = i)
    CashPosition = BaseBalance - StockPosition - BondPosition
    TotalPosition = StockPosition + BondPosition + CashPosition

    Operating_format = [BM.index[i], BaseBalance, StockPosition, BondPosition, CashPosition, TotalPosition]
    Operation_Log.append(Operating_format)
    


Operation_Log
print("time :", time.time() - start)  # 현재시각 - 시작시간 = 실행 시간
