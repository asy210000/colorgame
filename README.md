
# Discord Color Game Bot

A Discord bot for managing a color-based betting game. Players can place bets on colors, and the bot handles payouts, leaderboards, and more. Built using `discord.py` and MongoDB for data storage.

## Features

- **Color Betting**: Players can bet on colors (red, purple, pink, orange, blue, green) with customizable limits.
- **Leaderboard**: Tracks user balances and displays a leaderboard.
- **Admin Controls**: Admins can adjust balances, reset data, and manage betting sessions.
- **Redeemable Rewards**: Players can redeem coins for rewards like Discord Nitro, Steam Gift Cards, or GCash credits.
- **Bet History**: Keeps a log of all bets and roll results.
- **Session Management**: Admins can open and close betting sessions with a countdown timer.

## Commands

### User Commands
- `.bet <amount> <color>`: Place a bet on a color.
- `.cancel_bet`: Cancel your most recent bet.
- `.balance [user]`: Check your or another user's balance.
- `.view_bets`: View your active bets.
- `.gift <user> <amount>`: Gift coins to another user.
- `.redeem <reward_type> <amount>`: Redeem coins for rewards (e.g., Nitro, Steam, GCash).
- `.withdraw <amount>`: Request a withdrawal of coins (requires admin approval).

### Admin Commands
- `.open_bets`: Open betting for a session.
- `.close_bets`: Close betting and display results.
- `.roll_colors <color1> <color2> <color3>`: Roll the colors for the current betting session.
- `.give_coins <user> <amount>`: Give coins to a user.
- `.adjust_balance <user> <amount>`: Adjust a user's balance.
- `.reset_balances`: Reset all user balances to zero.
- `.reset_history`: Clear the roll history.
- `.reset_profits`: Reset profit tracking data.
- `.view_profits`: View total profits and coins given.

## Setup

### Prerequisites
- **Python 3.8+**: Ensure Python is installed.
- **MongoDB**: Set up a MongoDB database and get the connection URI.
- **Discord Bot Token**: Create a bot on the [Discord Developer Portal](https://discord.com/developers/applications) and get the bot token.

### Installation
1. Clone the repository:
   ```bash
   git clone https://github.com/your-username/discord-color-game-bot.git
   cd discord-color-game-bot
   ```
2. Install dependencies:
   ```bash
   pip install -r requirements.txt
   ```
3. Set up environment variables:
   - Create a `.env` file in the root directory.
   - Add the following variables:
     ```
     MONGODB_URI=your_mongodb_uri
     DISCORD_TOKEN=your_discord_bot_token
     ```

4. Run the bot:
   ```bash
   python bot.py
   ```

## Configuration
- **Allowed Channels**: Update the `allowed_channels` list in the `ColorGame` class to specify where commands can be used.
- **Bet Limits**: Adjust `bet_limit` and `session_bet_limit` in the `ColorGame` class.
- **Rewards**: Modify the `rewards` dictionary in the `redeem_request` command to add or change redeemable rewards.

## Contributing

Contributions are welcome! If you find a bug or have a feature request, please open an issue or submit a pull request.

## License

This project is licensed under the MIT License. See the LICENSE file for details.
