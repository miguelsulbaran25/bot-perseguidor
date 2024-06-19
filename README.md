
//+------------------------------------------------------------------+
//|                                             medias+stocatick.mq5 |
//|                                  Copyright 2024, MetaQuotes Ltd. |
//|                                             https://www.mql5.com |
//+------------------------------------------------------------------+
#property copyright "Copyright 2024, MetaQuotes Ltd."
#property link      "https://www.mql5.com"
#property version   "1.02"

#include <Trade\Trade.mqh>  // Incluye la biblioteca de trading

// Parámetros de entrada del Expert Advisor
input double InpLots = 0.01;            // Tamaño de lote inicial
input double InpLotsFactor = 2;         // Factor de incremento de lotes
input bool   InpEnableLotFactor = true; // Activar/desactivar el factor de incremento de lote
input double MaxLotSize = 1.0;          // Tamaño máximo del lote
input int    InpTakeProfit = 100;       // Take Profit en puntos
input int    InpStopLoss = 100;         // Stop Loss en puntos
input int    InpMagicNumber = 111;      // Número mágico para identificar las órdenes del EA
input bool   EnableBreakEven = true;    // Activar/desactivar Break Even
input int    BreakEvenLevel = 50;       // Nivel de activación de Break Even en puntos
input int    BreakEvenOffset = 10;      // Offset de Break Even en puntos
input bool   EnableTrailingStop = true; // Activar/desactivar Trailing Stop

CTrade trade;                           // Objeto para operaciones de trading
bool isTradeAllowed = true;             // Variable para controlar si se permite abrir una nueva operación

//+------------------------------------------------------------------+
//| Expert initialization function                                   |
//+------------------------------------------------------------------+
int OnInit()
{
   trade.SetExpertMagicNumber(InpMagicNumber);  // Establece el número mágico para las operaciones
   return(INIT_SUCCEEDED);                      // Inicialización exitosa
}

//+------------------------------------------------------------------+
//| Expert deinitialization function                                 |
//+------------------------------------------------------------------+
void OnDeinit(const int reason)
{
   // Aquí puedes añadir alguna limpieza si es necesario
}

//+------------------------------------------------------------------+
//| Expert tick function                                             |
//+------------------------------------------------------------------+
void OnTick()
{
   // Manejamos Break Even y Trailing Stop para cada posición
   ManageBreakEven();
   ManageTrailingStop();

   if(isTradeAllowed)  // Verifica si se permite abrir una nueva operación
   {
      double ask = SymbolInfoDouble(_Symbol, SYMBOL_ASK);  // Obtiene el precio ask actual
      double tp = ask + InpTakeProfit * _Point;            // Calcula el nivel de Take Profit
      double sl = ask - InpStopLoss * _Point;              // Calcula el nivel de Stop Loss

      ask = NormalizeDouble(ask, _Digits);  // Normaliza el precio ask
      tp = NormalizeDouble(tp, _Digits);    // Normaliza el Take Profit
      sl = NormalizeDouble(sl, _Digits);    // Normaliza el Stop Loss

      if(trade.Buy(InpLots, _Symbol, ask, sl, tp))  // Intenta abrir una operación de compra
      {
         isTradeAllowed = false;  // Deshabilita la apertura de nuevas operaciones hasta que se cumpla alguna condición en OnTradeTransaction
      }
   }
}

//+------------------------------------------------------------------+
//| Trade transaction handler                                        |
//+------------------------------------------------------------------+
void OnTradeTransaction(const MqlTradeTransaction& trans, const MqlTradeRequest& request, const MqlTradeResult& result)
{
   if(trans.type == TRADE_TRANSACTION_DEAL_ADD)  // Verifica si se ha añadido una nueva operación
   {
      CDealInfo deal;
      deal.Ticket(trans.deal);  // Obtiene la información de la operación
      HistorySelect(TimeCurrent() - PeriodSeconds(PERIOD_D1), TimeCurrent() + 10);  // Selecciona el historial reciente

      // Verifica si la operación fue realizada por este EA y si es de este símbolo
      if(deal.Magic() == InpMagicNumber && deal.Symbol() == _Symbol)
      {
         if(deal.Entry() == DEAL_ENTRY_OUT)  // Verifica si la operación ha sido cerrada
         {
            Print(__FUNCTION__, "> cerrar posicion #", trans.position);  // Imprime información sobre el cierre de la posición

            if(deal.Profit() > 0)  // Si la operación fue rentable
            {
               isTradeAllowed = true;  // Permite abrir una nueva operación
            }
            else  // Si la operación fue con pérdida
            {
               double lots = InpEnableLotFactor ? NormalizeDouble(deal.Volume() * InpLotsFactor, 2) : InpLots;
               lots = MathMin(lots, MaxLotSize);  // Asegura que el tamaño de lote no exceda el máximo permitido

               if(deal.DealType() == DEAL_TYPE_BUY)  // Si la operación cerrada fue una compra
               {
                  double ask = SymbolInfoDouble(_Symbol, SYMBOL_ASK);  // Obtiene el precio ask actual
                  double tp = ask + InpTakeProfit * _Point;            // Calcula el nivel de Take Profit
                  double sl = ask - InpStopLoss * _Point;              // Calcula el nivel de Stop Loss

                  ask = NormalizeDouble(ask, _Digits);  // Normaliza el precio ask
                  tp = NormalizeDouble(tp, _Digits);    // Normaliza el Take Profit
                  sl = NormalizeDouble(sl, _Digits);    // Normaliza el Stop Loss

                  trade.Buy(lots, _Symbol, ask, sl, tp);  // Abre una nueva operación de compra con el tamaño de lote ajustado
               }
               else if(deal.DealType() == DEAL_TYPE_SELL)  // Si la operación cerrada fue una venta
               {
                  double bid = SymbolInfoDouble(_Symbol, SYMBOL_BID);  // Obtiene el precio bid actual
                  double tp = bid - InpTakeProfit * _Point;            // Calcula el nivel de Take Profit
                  double sl = bid + InpStopLoss * _Point;              // Calcula el nivel de Stop Loss

                  bid = NormalizeDouble(bid, _Digits);  // Normaliza el precio bid
                  tp = NormalizeDouble(tp, _Digits);    // Normaliza el Take Profit
                  sl = NormalizeDouble(sl, _Digits);    // Normaliza el Stop Loss

                  trade.Sell(lots, _Symbol, bid, sl, tp);  // Abre una nueva operación de venta con el tamaño de lote ajustado
               }
            }
         }
      }
   }
}

//+------------------------------------------------------------------+
//| Manage Break Even                                                |
//+------------------------------------------------------------------+
void ManageBreakEven()
{
   for(int i = PositionsTotal() - 1; i >= 0; i--)
   {
      ulong ticket = PositionGetTicket(i);
      if(PositionGetInteger(POSITION_MAGIC) == InpMagicNumber && PositionGetString(POSITION_SYMBOL) == _Symbol)
      {
         double openPrice = PositionGetDouble(POSITION_PRICE_OPEN);
         double currentPrice = SymbolInfoDouble(_Symbol, PositionGetInteger(POSITION_TYPE) == POSITION_TYPE_BUY ? SYMBOL_BID : SYMBOL_ASK);
         double sl = PositionGetDouble(POSITION_SL);
         double tp = PositionGetDouble(POSITION_TP);

         // Break Even
         if(EnableBreakEven)
         {
            if(PositionGetInteger(POSITION_TYPE) == POSITION_TYPE_BUY && (currentPrice - openPrice) >= BreakEvenLevel * _Point)
            {
               double newSL = NormalizeDouble(openPrice + BreakEvenOffset * _Point, _Digits);
               if(sl < newSL)
               {
                  trade.PositionModify(ticket, newSL, tp);
               }
            }
            else if(PositionGetInteger(POSITION_TYPE) == POSITION_TYPE_SELL && (openPrice - currentPrice) >= BreakEvenLevel * _Point)
            {
               double newSL = NormalizeDouble(openPrice - BreakEvenOffset * _Point, _Digits);
               if(sl > newSL)
               {
                  trade.PositionModify(ticket, newSL, tp);
               }
            }
         }
      }
   }
}

//+------------------------------------------------------------------+
//| Manage Trailing Stop                                             |
//+------------------------------------------------------------------+
void ManageTrailingStop()
{
   for(int i = PositionsTotal() - 1; i >= 0; i--)
   {
      ulong ticket = PositionGetTicket(i);
      if(PositionGetInteger(POSITION_MAGIC) == InpMagicNumber && PositionGetString(POSITION_SYMBOL) == _Symbol)
      {
         double openPrice = PositionGetDouble(POSITION_PRICE_OPEN);
         double currentPrice = SymbolInfoDouble(_Symbol, PositionGetInteger(POSITION_TYPE) == POSITION_TYPE_BUY ? SYMBOL_BID : SYMBOL_ASK);
         double sl = PositionGetDouble(POSITION_SL);
         double tp = PositionGetDouble(POSITION_TP);

         // Trailing Stop
         if(EnableTrailingStop)
         {
            if(PositionGetInteger(POSITION_TYPE) == POSITION_TYPE_BUY)
            {
               double newSL = NormalizeDouble(currentPrice - InpStopLoss * _Point, _Digits);
               if(currentPrice > openPrice && sl < newSL)
               {
                  trade.PositionModify(ticket, newSL, tp);
               }
            }
            else if(PositionGetInteger(POSITION_TYPE) == POSITION_TYPE_SELL)
            {
               double newSL = NormalizeDouble(currentPrice + InpStopLoss * _Point, _Digits);
               if(currentPrice < openPrice && sl > newSL)
               {
                  trade.PositionModify(ticket, newSL, tp);
               }
            }
         }
      }
   }
}
