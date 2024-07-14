import asyncio
import json
from deriv_api import DerivAPI
import random
import time

class DerivTradingBot:
    def __init__(self, app_id, api_token):
        self.app_id = app_id
        self.api_token = api_token
        self.api = None
        self.max_trade_amount = 1000  # Maximum trade amount in USD
        self.min_trade_amount = 1  # Minimum trade amount in USD
        self.target_profit_multiplier = 2  # Target 2x profit instead of 10x
        self.max_consecutive_losses = 3  # Max number of losses before stopping
        self.recovery_multiplier = 2  # Multiply stake by 2 after a loss

    async def connect(self):
        self.api = DerivAPI(app_id=self.app_id)
        await self.api.authorize(self.api_token)

    async def get_balance(self):
        balance = await self.api.balance()
        return balance['balance']['balance']

    async def analyze_market(self, symbol, duration):
        history = await self.api.ticks_history(
            symbol,
            end="latest",
            count=100,
            style="candles",
            granularity=60
        )
        
        candles = history['candles']
        closes = [float(candle['close']) for candle in candles]
        
        trend = 'up' if closes[-1] > closes[0] else 'down'
        
        last_digits = [int(str(close)[-1]) for close in closes]
        even_count = sum(1 for digit in last_digits if digit % 2 == 0)
        odd_count = len(last_digits) - even_count
        
        digit_bias = 'even' if even_count > odd_count else 'odd'
        
        # Additional analysis could be added here
        
        return {
            'trend': trend,
            'digit_bias': digit_bias
        }

    async def place_trade(self, symbol, contract_type, amount, duration):
        try:
            proposal = await self.api.proposal(
                contract_type=contract_type,
                amount=amount,
                duration=duration,
                duration_unit='m',
                symbol=symbol,
                currency='USD'
            )
            
            if 'proposal' in proposal:
                buy = await self.api.buy(proposal['proposal']['id'], amount)
                return buy
            else:
                print(f"Error in proposal: {proposal}")
                return None
        except Exception as e:
            print(f"Error placing trade: {e}")
            return None

    async def monitor_trade(self, contract_id):
        while True:
            contract = await self.api.proposal_open_contract(contract_id=contract_id)
            if contract['proposal_open_contract']['is_sold'] == 1:
                profit = contract['proposal_open_contract']['profit']
                return profit
            await asyncio.sleep(1)

    async def run(self):
        await self.connect()
        
        consecutive_losses = 0
        current_stake = self.min_trade_amount

        while True:
            balance = await self.get_balance()
            print(f"Current balance: ${balance}")

            symbol = "R_10"  # Example: Random index
            duration = 1  # Example: 1 minute duration

            analysis = await self.analyze_market(symbol, duration)
            
            # Determine trade amount
            trade_amount = min(current_stake, self.max_trade_amount)

            # Choose between EVEN and ODD based on analysis
            contract_type = 'DIGITEVEN' if analysis['digit_bias'] == 'even' else 'DIGITODD'

            trade = await self.place_trade(symbol, contract_type, trade_amount, duration)
            
            if trade and 'buy' in trade:
                contract_id = trade['buy']['contract_id']
                profit = await self.monitor_trade(contract_id)
                print(f"Trade completed. Profit: ${profit}")

                if profit > 0:
                    consecutive_losses = 0
                    current_stake = self.min_trade_amount
                else:
                    consecutive_losses += 1
                    current_stake *= self.recovery_multiplier

                if consecutive_losses >= self.max_consecutive_losses:
                    print("Max consecutive losses reached. Stopping trading.")
                    break

            else:
                print("Failed to place trade")

            await asyncio.sleep(60)  # Wait for 1 minute before next trade

if __name__ == "__main__":
    app_id = "YOUR_APP_ID"
    api_token = "YOUR_API_TOKEN"
    
    bot = DerivTradingBot(app_id, api_token)
    asyncio.run(bot.run())
