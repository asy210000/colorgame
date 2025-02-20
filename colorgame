import asyncio
import datetime
import os

from pymongo import MongoClient, UpdateOne
from discord.ext import commands
import discord

# MongoDB setup
cluster = MongoClient(os.getenv('MONGODB_URI'))
db = cluster["discord"]
users_collection = db["users"]
bets_collection = db["bets"]
roll_history_collection = db["roll_history"]
given_coins_collection = db["given_coins"]
lost_bets_collection = db["lost_bets"]
redeemed_coins_collection = db["redeemed_coins"]

# Emoji constants
EMOJI_PESO_COIN = "Replace with own emoji"
EMOJI_RED = "Replace with own emoji"
EMOJI_PURPLE = "Replace with own emoji"
EMOJI_PINK = "Replace with own emoji"
EMOJI_ORANGE = "Replace with own emoji"
EMOJI_BLUE = "Replace with own emoji"
EMOJI_GREEN = "Replace with own emoji"


class ColorGame(commands.Cog):
    def __init__(self, client):
        self.client = client
        self.allowed_channels = ["replace with your channel ID's]
        self.colors = {
            'red': EMOJI_RED,
            'purple': EMOJI_PURPLE,
            'pink': EMOJI_PINK,
            'orange': EMOJI_ORANGE,
            'blue': EMOJI_BLUE,
            'green': EMOJI_GREEN
        }
        self.betting_open = False
        self.bet_limit = 500  # Max bet per transaction
        self.session_bet_limit = 500  # Max total bet per user per session
        self.user_session_bets = {}  # Tracks bets per user per session
        self.timer_active = False
        self.betting_timer_duration = 30

    def in_allowed_channels():
        async def predicate(ctx):
            return ctx.channel.id in ctx.cog.allowed_channels

        return commands.check(predicate)

    def is_specific_user():
        async def predicate(ctx):
            return ctx.author.id == 879766888513687602

        return commands.check(predicate)

    @commands.command()
    @in_allowed_channels()
    async def cancel_bet(self, ctx, bet_id: str = None):
        if not self.betting_open and not ctx.author.guild_permissions.manage_guild:
            await ctx.reply("Betting is currently closed. Admin privileges required to cancel bets at this time.")
            return

        if bet_id and ctx.author.guild_permissions.manage_guild:
            bet = bets_collection.find_one({"_id": bet_id})
            if not bet:
                await ctx.reply("No bet found with that ID.")
                return
            user = self.client.get_user(bet['user_id'])
            action_msg = f"Bet of {bet['amount']} coins on {bet['color']} by {user.display_name if user else 'Unknown User'} has been canceled by admin."
        else:
            user_bets = list(bets_collection.find({"user_id": ctx.author.id}).sort("date", -1))
            if not user_bets:
                await ctx.reply("You do not have any active bets to cancel.")
                return
            bet = user_bets[0]
            action_msg = f"Your recent bet of {EMOJI_PESO_COIN}{bet['amount']} peso coins on {bet['color']} has been canceled."

        bets_collection.delete_one({"_id": bet["_id"]})
        users_collection.update_one({"_id": bet['user_id']}, {"$inc": {"balance": bet["amount"]}})
        await ctx.send(action_msg)

    @commands.command()
    @in_allowed_channels()
    @is_specific_user()
    async def set_bet_limit(self, ctx, limit: int):
        if limit < 1:
            await ctx.reply("Bet limit must be at least 1 peso coin.")
            return
        self.bet_limit = limit
        await ctx.reply(f"Betting limit set to {EMOJI_PESO_COIN}{limit} peso coins.")

    @commands.command(aliases=['viewbets', 'viewbet'])
    @in_allowed_channels()
    async def view_bets(self, ctx):
        user_bets = list(bets_collection.find({"user_id": ctx.author.id}).sort("date", -1))
        if not user_bets:
            await ctx.reply(embed=discord.Embed(description="You have no active bets.", color=0xffcba4))
            return

        embeds = []
        embed = discord.Embed(title="Active Bets", color=0xffcba4)
        field_count = 0

        for bet in user_bets:
            if field_count == 25:
                embeds.append(embed)
                embed = discord.Embed(title="Active Bets (cont.)", color=0xffcba4)
                field_count = 0

            color_emoji = self.colors.get(bet['color'], "Unknown Color")
            embed.add_field(
                name=f"{color_emoji} Bet",
                value=f"Amount: {bet['amount']} coins\nPlaced on: {bet['date'].strftime('%Y-%m-%d %H:%M:%S')}",
                inline=False
            )
            field_count += 1

        embeds.append(embed)

        for emb in embeds:
            await ctx.reply(embed=emb)

    @commands.command(aliases=['give'])
    @in_allowed_channels()
    @is_specific_user()
    async def give_coins(self, ctx, member: discord.Member, amount: int):
        if amount < 0:
            embed = discord.Embed(description="You cannot give a negative amount.", color=0xffcba4)
            await ctx.send(embed=embed)
            return

        users_collection.update_one({"_id": member.id}, {"$inc": {"balance": amount}}, upsert=True)
        embed = discord.Embed(
            title="Coins Given",
            description=f"Added {EMOJI_PESO_COIN}{amount} peso coins to {member.display_name}'s balance.",
            color=0xffcba4
        )
        await ctx.send(embed=embed)

        given_coins_collection.insert_one({
            "user_id": member.id,
            "amount": amount,
            "date": datetime.datetime.now()
        })

    @commands.command(aliases=['open'])
    @commands.has_role('admin')
    @in_allowed_channels()
    @is_specific_user()
    async def open_bets(self, ctx):
        if self.betting_open:
            await ctx.send("Betting is already open.")
            return

        self.betting_open = True
        self.user_session_bets = {}
        self.timer_active = True
        color_emojis = ' '.join(self.colors.values())
        message = await ctx.send(
            f"**Betting is now OPEN! Place your bets!**\n{color_emojis}\nClosing in: {self.betting_timer_duration} seconds")

        for remaining in range(self.betting_timer_duration, 0, -5):
            await asyncio.sleep(5)
            if not self.timer_active:
                break
            await message.edit(
                content=f"**Betting is now OPEN! Place your bets!**\n{color_emojis}\nClosing in: {remaining - 5} seconds")
            if remaining <= 5:
                await message.edit(content=f"**Betting is now OPEN! Place your bets!**\n{color_emojis}\nClosed!")
                break

        if self.timer_active:
            await self.close_bets(ctx)

    @commands.command(aliases=['close'])
    @commands.has_role('admin')
    @in_allowed_channels()
    async def close_bets(self, ctx):
        if not self.betting_open:
            await ctx.reply("Betting is already closed.")
            return

        self.betting_open = False
        self.timer_active = False

        all_bets = list(bets_collection.find().sort("date", -1))
        if not all_bets:
            await ctx.reply(embed=discord.Embed(description="No bets were placed.", color=0xffcba4))
            return

        aggregated_bets = {}
        for bet in all_bets:
            key = (bet['user_id'], bet['color'])
            aggregated_bets[key] = aggregated_bets.get(key, 0) + bet['amount']

        pages = []
        page = discord.Embed(title="⚠ ALL BETS ARE NOW CLOSED ⚠ Rolling Soon.", description="**Active Bets:**",
                             color=0xffcba4)
        field_count = 0
        for (user_id, color), total_amount in aggregated_bets.items():
            user = await ctx.guild.fetch_member(user_id)
            user_name = user.display_name if user else 'Unknown User'
            user_info = f"{user_name} (*{user_id}*)" if user else "Unknown User (*Unknown ID*)"
            color_emoji = self.colors[color] if color in self.colors else "Unknown Color"
            page.add_field(name=user_info, value=f"{total_amount} on {color_emoji}", inline=False)
            field_count += 1

            if field_count == 25:
                pages.append(page)
                page = discord.Embed(title="Active Bets (cont.)", color=0xffcba4)
                field_count = 0

        if field_count > 0:
            pages.append(page)

        message = await ctx.send(embed=pages[0])
        if len(pages) > 1:
            await message.add_reaction("⬅️")
            await message.add_reaction("➡️")

        current_page = 0

        def check(reaction, user):
            return user == ctx.author and str(reaction.emoji) in ["⬅️", "➡️"] and reaction.message.id == message.id

        while True:
            try:
                reaction, user = await ctx.bot.wait_for('reaction_add', timeout=60.0, check=check)
                if str(reaction.emoji) == "➡️" and current_page < len(pages) - 1:
                    current_page += 1
                    await message.edit(embed=pages[current_page])
                elif str(reaction.emoji) == "⬅️" and current_page > 0:
                    current_page -= 1
                    await message.edit(embed=pages[current_page])
                await message.remove_reaction(reaction, user)
            except asyncio.TimeoutError:
                break

        await message.clear_reactions()

    @commands.command(aliases=['bet'])
    @in_allowed_channels()
    async def start_bet(self, ctx, amount: int = None, color: str = None):
        if not self.betting_open:
            await ctx.reply(embed=discord.Embed(description="Betting is currently closed.", color=0xffcba4))
            return
        if amount is None or amount <= 0 or amount > self.bet_limit:
            await ctx.reply(embed=discord.Embed(
                description=f"Invalid amount. Bet amount should be between {EMOJI_PESO_COIN}5 and {EMOJI_PESO_COIN}{self.bet_limit} peso coins.",
                color=0xffcba4))
            return
        if color.lower() not in self.colors:
            await ctx.reply("Specify a valid color: red, purple, pink, orange, blue, green.")
            return

        user_total_bets = self.user_session_bets.get(ctx.author.id, 0) + amount
        if user_total_bets > self.session_bet_limit:
            await ctx.reply(
                f"Total betting limit per session is {EMOJI_PESO_COIN}{self.session_bet_limit}. Your total bets exceed this limit.")
            return

        new_balance = users_collection.find_one_and_update(
            {"_id": ctx.author.id, "balance": {"$gte": amount}},
            {"$inc": {"balance": -amount}},
            return_document=True
        )
        if not new_balance:
            await ctx.reply(embed=discord.Embed(description="Insufficient balance.", color=0xffcba4))
            return

        self.user_session_bets[ctx.author.id] = user_total_bets
        bets_collection.insert_one(
            {"user_id": ctx.author.id, "color": color.lower(), "amount": amount, "date": datetime.datetime.now()})
        embed = discord.Embed(title="Bet Placed",
                              description=f"{ctx.author.display_name} bets {EMOJI_PESO_COIN}`{amount}` peso coins on **{self.colors[color.lower()]}**.",
                              color=0xffcba4)
        embed.set_footer(text=f"New Balance: {new_balance['balance']} peso coins")
        await ctx.reply(embed=embed)

    @start_bet.error
    async def start_bet_error(self, ctx, error):
        if isinstance(error, commands.CommandError):
            await ctx.reply("To place a bet, do `.bet <amount> <color>`")

    @commands.command(aliases=['roll'])
    @commands.has_role('admin')
    @in_allowed_channels()
    async def roll_colors(self, ctx, *results):
        if not all(result in self.colors for result in results) or len(results) != 3:
            await ctx.send("Invalid results. Enter three colors from red, purple, pink, orange, blue, green.")
            return

        emoji_message = " ".join([self.colors[color] for color in results])
        await ctx.send(emoji_message)

        roll_entry = {
            "results": results,
            "date": datetime.datetime.now()
        }
        roll_history_collection.insert_one(roll_entry)

        all_bets = list(bets_collection.find())
        if not all_bets:
            await ctx.send("No bets to roll.")
            return

        aggregated_bets = {}
        for bet in all_bets:
            user_id = bet['user_id']
            color = bet['color'].lower()
            amount = bet['amount']
            if (user_id, color) not in aggregated_bets:
                aggregated_bets[(user_id, color)] = 0
            aggregated_bets[(user_id, color)] += amount

        results_messages = []
        for (user_id, color), total_amount in aggregated_bets.items():
            color_hits = sum(1 for result in results if result == color)
            user = await ctx.guild.fetch_member(user_id)
            user_mention = user.mention if user else "User not found"
            multiplier = color_hits + 1 if color_hits > 0 else 0
            payout = total_amount * multiplier

            if color_hits > 0:
                results_messages.append(
                    f"{user_mention} wins {EMOJI_PESO_COIN}{payout} for {color_hits} hits on {self.colors[color]}")
                users_collection.update_one({"_id": user_id}, {"$inc": {"balance": payout}})
            else:
                results_messages.append(
                    f"{user_mention} loses {EMOJI_PESO_COIN}{total_amount} on {self.colors[color]}")
                lost_bets_collection.insert_one({
                    "user_id": user_id,
                    "amount": total_amount,
                    "date": datetime.datetime.now()
                })

        if results_messages:
            for i in range(0, len(results_messages), 25):
                await ctx.send("\n".join(results_messages[i:i + 25]))
        else:
            await ctx.send("No results to process this time.")

        bets_collection.delete_many({})

    @commands.command()
    @in_allowed_channels()
    async def leaderboard(self, ctx):
        top_users = list(users_collection.find().sort("balance", -1))
        if not top_users:
            await ctx.reply("Leaderboard is empty.")
            return

        def get_embed_for_users(users, start_index):
            embed = discord.Embed(title=f"{EMOJI_PESO_COIN}peso Coins Leaderboard{EMOJI_PESO_COIN}",
                                  color=0xffcba4)
            for index, user in enumerate(users, start=start_index):
                member = ctx.guild.get_member(user["_id"])
                name = member.display_name if member else 'Unknown User'
                embed.add_field(name=f"{index}. {name}", value=f"{user['balance']} coins", inline=False)
            return embed

        chunks = [top_users[i:i + 10] for i in range(0, len(top_users), 10)]
        pages = [get_embed_for_users(chunk, i * 10 + 1) for i, chunk in enumerate(chunks)]

        message = await ctx.reply(embed=pages[0])
        await message.add_reaction('⬅️')
        await message.add_reaction('➡️')

        def check(reaction, user):
            return user == ctx.author and str(reaction.emoji) in ['⬅️', '➡️'] and reaction.message.id == message.id

        current_page = 0
        while True:
            try:
                reaction, user = await self.client.wait_for('reaction_add', timeout=60.0, check=check)
                if str(reaction.emoji) == '➡️' and current_page < len(pages) - 1:
                    current_page += 1
                    await message.edit(embed=pages[current_page])
                    await message.remove_reaction(reaction, user)
                elif str(reaction.emoji) == '⬅️' and current_page > 0:
                    current_page -= 1
                    await message.edit(embed=pages[current_page])
                    await message.remove_reaction(reaction, user)
                else:
                    await message.remove_reaction(reaction, user)
            except asyncio.TimeoutError:
                await message.clear_reactions()
                break

    @commands.command(aliases=['withdraw'])
    @in_allowed_channels()
    async def withdraw_request(self, ctx, amount: int):
        if amount <= 0:
            embed = discord.Embed(description="You cannot withdraw a negative or zero amount.", color=0xffcba4)
            await ctx.send(embed=embed)
            return

        user_data = users_collection.find_one({"_id": ctx.author.id})
        if not user_data or user_data['balance'] < amount:
            embed = discord.Embed(description="Insufficient balance to make this withdrawal.", color=0xffcba4)
            await ctx.send(embed=embed)
            return

        embed = discord.Embed(description="Waiting approval from admins", color=0xffcba4)
        await ctx.reply(embed=embed)

        admin_channel = self.client.get_channel(1256432375626207283)
        message = await admin_channel.send(
            f"Withdrawal request from {ctx.author.display_name} for {EMOJI_PESO_COIN}{amount} peso coins. React with ✅ to approve.")
        await message.add_reaction('✅')

        def check(reaction, user):
            return user != self.client.user and str(reaction.emoji) == '✅' and any(
                role.name == 'admin' for role in user.roles)

        try:
            reaction, user = await self.client.wait_for('reaction_add', timeout=86400.0, check=check)
            new_balance = user_data['balance'] - amount
            users_collection.update_one({"_id": ctx.author.id}, {"$set": {"balance": new_balance}})
            embed = discord.Embed(
                title="Approved!",
                description=f"Updated balance for {ctx.author.display_name}: {user_data['balance']} - {amount} = {EMOJI_PESO_COIN}{new_balance} peso coins.",
                color=0xffcba4)
            await ctx.reply(embed=embed)
        except asyncio.TimeoutError:
            await admin_channel.send("No one responded to the withdrawal request in time. It has been canceled.")

    @commands.command()
    @in_allowed_channels()
    @is_specific_user()
    async def adjust_balance(self, ctx, member: discord.Member, adjustment: int):
        if adjustment == 0:
            embed = discord.Embed(description="No adjustment made. Please specify a non-zero amount.", color=0xffcba4)
            await ctx.send(embed=embed)
            return

        user_data = users_collection.find_one({"_id": member.id})
        if not user_data:
            users_collection.insert_one({"_id": member.id, "balance": 0})
            current_balance = 0
        else:
            current_balance = user_data['balance']

        new_balance = current_balance + adjustment
        if new_balance < 0:
            embed = discord.Embed(
                description=f"Adjustment failed. The balance cannot go negative. {member.display_name}'s current balance is {EMOJI_PESO_COIN}{current_balance} peso coins.",
                color=0xffcba4)
            await ctx.send(embed=embed)
            return

        users_collection.update_one({"_id": member.id}, {"$set": {"balance": new_balance}})
        embed = discord.Embed(title="Balance Adjustment",
                              description=f"{member.display_name}'s balance adjusted by {adjustment} peso coins. New balance: {EMOJI_PESO_COIN}{new_balance} peso coins.",
                              color=0xffcba4)
        await ctx.send(embed=embed)

    @commands.command()
    @in_allowed_channels()
    @is_specific_user()
    async def mass_giveaway(self, ctx, amount: int, *members: discord.Member):
        if amount <= 0:
            await ctx.send("Amount must be greater than zero.")
            return

        if not members:
            await ctx.send("No members specified.")
            return

        bulk_operations = [
            UpdateOne({"_id": member.id}, {"$inc": {"balance": amount}}, upsert=True)
            for member in members
        ]

        try:
            if bulk_operations:
                loop = asyncio.get_running_loop()
                await loop.run_in_executor(None, lambda: users_collection.bulk_write(bulk_operations))
                result_message = "\n".join(
                    [f"{member.mention} has received {EMOJI_PESO_COIN}{amount} peso coins." for member in members])
                await ctx.send(f"Updated balances for {len(members)} members:\n{result_message}")

                await loop.run_in_executor(None, lambda: given_coins_collection.insert_many([
                    {"user_id": member.id, "amount": amount, "date": datetime.datetime.now()}
                    for member in members
                ]))

        except Exception as e:
            await ctx.send(f"Failed to update balances due to an error: {e}")

    @commands.command(aliases=['gift'])
    @in_allowed_channels()
    async def gift_coins(self, ctx, recipient: discord.Member, amount: int):
        if amount <= 0:
            await ctx.send(embed=discord.Embed(description="You cannot gift a negative or zero amount of coins.",
                                               color=0xffcba4))
            return

        try:
            user_data = users_collection.find_one({"_id": ctx.author.id})
            if not user_data or user_data.get('balance', 0) < amount:
                await ctx.send(embed=discord.Embed(description="Insufficient balance to gift this amount of coins.",
                                                   color=0xffcba4))
                return

            new_giver_balance = user_data['balance'] - amount
            users_collection.update_one({"_id": ctx.author.id}, {"$set": {"balance": new_giver_balance}})

            recipient_data = users_collection.find_one({"_id": recipient.id})
            if not recipient_data:
                users_collection.insert_one({"_id": recipient.id, "balance": amount})
            else:
                new_recipient_balance = recipient_data['balance'] + amount
                users_collection.update_one({"_id": recipient.id}, {"$set": {"balance": new_recipient_balance}})

            embed = discord.Embed(
                description=f"You have gifted {EMOJI_PESO_COIN}`{amount}` peso coins to {recipient.display_name}.",
                color=0xffcba4)
            embed.set_footer(text=f"New Balance: {user_data['balance']} - {amount} = {new_giver_balance}")
            await ctx.reply(embed=embed)

        except Exception as e:
            await ctx.send(embed=discord.Embed(description=f"An error occurred while processing the command: {str(e)}",
                                               color=0xffcba4))
            print(f"Error in gift_coins command: {str(e)}")

    @commands.command(aliases=['bal'])
    @in_allowed_channels()
    async def balance(self, ctx, member: discord.Member = None):
        member = member or ctx.author
        try:
            user_data = users_collection.find_one({"_id": member.id})
            if not user_data:
                users_collection.insert_one({"_id": member.id, "balance": 0})
                balance = 0
            else:
                balance = user_data['balance']

            embed = discord.Embed(title=f"{member.display_name}'s Balance",
                                  description=f"{EMOJI_PESO_COIN}{balance} peso coins",
                                  color=0xffcba4)
            await ctx.reply(embed=embed)
        except Exception as e:
            await ctx.reply("Error retrieving balance. Please try again later.")
            print(f"Failed to retrieve balance: {e}")

    @commands.command()
    @in_allowed_channels()
    async def history(self, ctx):
        history_entries = list(roll_history_collection.find().sort("date", -1))
        if not history_entries:
            await ctx.send(embed=discord.Embed(description="No roll history available.", color=0xffcba4))
            return

        pages = []
        for i in range(0, len(history_entries), 10):
            embed = discord.Embed(title="Roll History", color=0xffcba4)
            for entry in history_entries[i:i + 10]:
                date_str = entry['date'].strftime('%Y-%m-%d %H:%M:%S')
                results_emojis = [self.colors[color] for color in entry['results']]
                results_str = ", ".join(results_emojis)
                embed.add_field(
                    name=date_str,
                    value=f"Colors: {results_str}",
                    inline=False
                )
            pages.append(embed)

        message = await ctx.send(embed=pages[0])
        if len(pages) > 1:
            await message.add_reaction("⬅️")
            await message.add_reaction("➡️")

        current_page = 0

        def check(reaction, user):
            return user == ctx.author and str(reaction.emoji) in ["⬅️", "➡️"] and reaction.message.id == message.id

        while True:
            try:
                reaction, user = await self.client.wait_for('reaction_add', timeout=60.0, check=check)
                if str(reaction.emoji) == "➡️" and current_page < len(pages) - 1:
                    current_page += 1
                    await message.edit(embed=pages[current_page])
                    await message.remove_reaction(reaction, user)
                elif str(reaction.emoji) == "⬅️" and current_page > 0:
                    current_page -= 1
                    await message.edit(embed=pages[current_page])
                    await message.remove_reaction(reaction, user)
                else:
                    await message.remove_reaction(reaction, user)
            except asyncio.TimeoutError:
                await message.clear_reactions()
                break

    @commands.command()
    @in_allowed_channels()
    @is_specific_user()
    async def reset_history(self, ctx):
        roll_history_collection.delete_many({})
        await ctx.send(embed=discord.Embed(description="Roll history has been reset.", color=0xffcba4))

    @commands.command()
    @in_allowed_channels()
    @is_specific_user()
    async def set_betting_timer(self, ctx, duration: int):
        if duration < 1:
            await ctx.reply("Betting timer duration must be at least 1 second.")
            return
        self.betting_timer_duration = duration
        await ctx.reply(f"Betting timer duration set to {duration} seconds.")

    @commands.command(aliases=['redeem'])
    @in_allowed_channels()
    async def redeem_request(self, ctx, reward_type: str = None, amount: int = None):
        rewards = {
            "nitro": {"cost": 500, "name": "Discord Nitro"},
            "steam": {"cost": 1, "name": "Steam Gift Card"},
            "gcash": {"cost": 1, "name": "GCash Credit"}
        }

        if reward_type is None or amount is None:
            embed = discord.Embed(
                title="What would you like to redeem?",
                description="**Steam Gift Card** -> do `.redeem steam <amount>`\n"
                            f"**Nitro**({EMOJI_PESO_COIN}500) -> do `.redeem nitro <amount>`\n"
                            "**GCash** -> do `.redeem gcash <amount>`",
                color=0xffcba4
            )
            await ctx.reply(embed=embed)
            return

        if reward_type.lower() not in rewards:
            await ctx.send("Please specify a valid reward type: nitro, steam, or gcash.")
            return

        if amount <= 0:
            embed = discord.Embed(description="You cannot redeem a negative or zero amount.", color=0xffcba4)
            await ctx.send(embed=embed)
            return

        reward = rewards[reward_type.lower()]
        total_cost = reward['cost'] * amount

        user_data = users_collection.find_one({"_id": ctx.author.id})
        if not user_data or user_data['balance'] < total_cost:
            embed = discord.Embed(description=f"Insufficient balance to redeem {amount} {reward['name']}.",
                                  color=0xffcba4)
            await ctx.send(embed=embed)
            return

        embed = discord.Embed(description="Waiting approval from admins", color=0xffcba4)
        await ctx.reply(embed=embed)

        admin_channel = self.client.get_channel(1256432375626207283)
        message = await admin_channel.send(
            f"Redemption request from {ctx.author.mention} for {amount} {reward['name']} worth {EMOJI_PESO_COIN}{total_cost} peso coins. React with ✅ to approve.")
        await message.add_reaction('✅')

        def check(reaction, user):
            return user != self.client.user and str(reaction.emoji) == '✅' and any(
                role.name == 'admin' for role in user.roles)

        try:
            reaction, user = await self.client.wait_for('reaction_add', timeout=86400.0, check=check)
            new_balance = user_data['balance'] - total_cost
            users_collection.update_one({"_id": ctx.author.id}, {"$set": {"balance": new_balance}})
            embed = discord.Embed(
                title="Approved!",
                description=f"{amount} {reward['name']} redeemed for {EMOJI_PESO_COIN}{total_cost} peso coins. New balance: {EMOJI_PESO_COIN}{new_balance}.",
                color=0xffcba4)
            await ctx.reply(embed=embed)

            redeemed_coins_collection.insert_one({
                "user_id": ctx.author.id,
                "amount": total_cost,
                "date": datetime.datetime.now()
            })
        except asyncio.TimeoutError:
            await admin_channel.send(
                f"No one responded to the redemption request from {ctx.author.display_name} in time. It has been canceled.")

    @commands.command()
    @in_allowed_channels()
    @is_specific_user()
    async def view_profits(self, ctx):
        total_given_result = given_coins_collection.aggregate([
            {"$group": {"_id": None, "total_given": {"$sum": "$amount"}}}
        ])
        total_given_amount = total_given_result.next().get('total_given', 0) if total_given_result.alive else 0

        total_lost_result = lost_bets_collection.aggregate([
            {"$group": {"_id": None, "total_lost": {"$sum": "$amount"}}}
        ])
        total_lost_amount = total_lost_result.next().get('total_lost', 0) if total_lost_result.alive else 0

        total_redeemed_result = redeemed_coins_collection.aggregate([
            {"$group": {"_id": None, "total_redeemed": {"$sum": "$amount"}}}
        ])
        total_redeemed_amount = total_redeemed_result.next().get('total_redeemed', 0) if total_redeemed_result.alive else 0

        total_profit = total_lost_amount - total_redeemed_amount

        embed = discord.Embed(
            title="Total Profits",
            description=(
                f"**Total Given:** {EMOJI_PESO_COIN}{total_given_amount} peso coins\n"
                f"**Profits:** {EMOJI_PESO_COIN}{total_profit} peso coins"
            ),
            color=0xffcba4
        )
        await ctx.reply(embed=embed)

    @commands.command()
    @in_allowed_channels()
    @is_specific_user()
    async def reset_balances(self, ctx):
        users_collection.update_many({}, {"$set": {"balance": 0}})
        await ctx.send(embed=discord.Embed(description="All user balances have been reset to zero.", color=0xffcba4))

    @commands.command()
    @in_allowed_channels()
    @is_specific_user()
    async def reset_profits(self, ctx):
        given_coins_collection.delete_many({})
        lost_bets_collection.delete_many({})
        redeemed_coins_collection.delete_many({})
        await ctx.send(embed=discord.Embed(description="Profit tracking data has been reset.", color=0xffcba4))

    @commands.command()
    @in_allowed_channels()
    @is_specific_user()
    async def adjust_total_given(self, ctx, amount: int):
        given_coins_collection.insert_one({
            "user_id": "manual_adjustment",
            "amount": amount,
            "date": datetime.datetime.now()
        })
        await ctx.send(embed=discord.Embed(description=f"Total given adjusted by {EMOJI_PESO_COIN}{amount}.", color=0xffcba4))

    @commands.command()
    @in_allowed_channels()
    @is_specific_user()
    async def adjust_profits(self, ctx, amount: int):
        lost_bets_collection.insert_one({
            "user_id": "manual_adjustment",
            "amount": amount,
            "date": datetime.datetime.now()
        })
        await ctx.send(embed=discord.Embed(description=f"Total profits adjusted by {EMOJI_PESO_COIN}{amount}.", color=0xffcba4))


async def setup(client):
    await client.add_cog(ColorGame(client))
