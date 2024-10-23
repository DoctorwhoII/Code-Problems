import asyncio
import os
import random
from datetime import datetime
import discord
from discord.app_commands import guilds
from discord.ext import commands
import sympy as sp
from typing import List  # Import List from typing
from discord import app_commands
import pytz
import json
import sqlite3

# Setting up intents
intents = discord.Intents.default()
intents.message_content = True
intents.members = True  # To interact with members
intents.messages = True
intents.guilds = True
intents.reactions = True
intents.voice_states = True  # Enable voice state events

# Bot Setup
bot = commands.Bot(command_prefix='$', intents=intents)

# On Ready Event
@bot.event
async def on_ready():
    print(f'We have logged in as {bot.user}')

    # Automatically sync command tree
    try:
        await bot.tree.sync()
        print('Command tree synced automatically.')
    except Exception as e:
        print(f'Error syncing command tree: {str(e)}')

# Syncing Command Tree (now optional)
@bot.command()
async def sync(ctx):
    if ctx.author.id == 888452653359697991:  # Replace with your Discord ID
        await bot.tree.sync()
        await ctx.send('Command tree synced manually.')
    else:
        await ctx.send('You must be the owner to use this command!')

# Ping Command
@bot.tree.command(name="ping", description="Shows the bot's latency.")
async def ping(interaction: discord.Interaction):
    latency = round(bot.latency * 1000)  # Latency in ms
    await interaction.response.send_message(f"Pong! Latency is {latency}ms.")

# Database setup
def get_db_name(guild_id):
    if guild_id == 1288424328819511376:  # Dialex
        return "Dialex_member_info.db"
    elif guild_id == 1289989965757026376:  # Nexus
        return "Nexus_member_info.db"
    return None

async def create_table(guild_id, interaction=None):
    db_name = get_db_name(guild_id)
    if db_name:
        conn = sqlite3.connect(db_name)
        cursor = conn.cursor()
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS members (
                MEMBER_ID INTEGER PRIMARY KEY,
                USERNAME TEXT,
                DISCRIMINATOR TEXT,
                JOIN_DATE TEXT,
                JOIN_TIME TEXT,
                ROLES TEXT,
                AVATAR_URL TEXT,
                WARNINGS INTEGER,
                PREMIUM_STATUS INTEGER,
                ToS INTEGER
            )
        ''')
        conn.commit()
        conn.close()

        if interaction:
            await interaction.response.send_message(f"Database for {interaction.guild.name} has been created or already exists.", ephemeral=True)

# Update database with member info
def update_member_info(guild_id, user_id, username, discriminator, roles, avatar_url, warnings, premium_status, tos):
    db_name = get_db_name(guild_id)
    if db_name:
        conn = sqlite3.connect(db_name)
        cursor = conn.cursor()
        cursor.execute('''
            INSERT INTO members (MEMBER_ID, USERNAME, DISCRIMINATOR, JOIN_DATE, JOIN_TIME, ROLES, AVATAR_URL, WARNINGS, PREMIUM_STATUS, ToS)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
            ON CONFLICT(MEMBER_ID) DO UPDATE SET
                USERNAME=excluded.USERNAME,
                DISCRIMINATOR=excluded.DISCRIMINATOR,
                JOIN_DATE=excluded.JOIN_DATE,
                JOIN_TIME=excluded.JOIN_TIME,
                ROLES=excluded.ROLES,
                AVATAR_URL=excluded.AVATAR_URL,
                WARNINGS=excluded.WARNINGS,
                PREMIUM_STATUS=excluded.PREMIUM_STATUS,
                ToS=excluded.ToS
        ''', (user_id, username, discriminator, datetime.now().strftime("%Y-%m-%d"),
              datetime.now().strftime("%H:%M:%S"), json.dumps(roles), avatar_url, warnings, premium_status, tos))
        conn.commit()
        conn.close()

# Command for accepting ToS (public)
@bot.tree.command(name="accept_tos", description="Accept the Terms of Service.")
async def accept_tos(interaction: discord.Interaction):
    user_id = interaction.user.id
    guild_id = interaction.guild.id

    # Update database with ToS acceptance
    update_member_info(guild_id, user_id, interaction.user.name, interaction.user.discriminator,
                       [role.name for role in interaction.user.roles], interaction.user.avatar.url, 0, 0, 1)

    await interaction.response.send_message("You have accepted the Terms of Service.", ephemeral=True)

# Command for updating member info (admin only)
@bot.tree.command(name="update_member_info", description="Update a member's information.")
@commands.has_permissions(administrator=True)
async def update_member_info_command(interaction: discord.Interaction, user: discord.User):
    guild_id = interaction.guild.id
    user_id = user.id

    # Get user information and update in the database
    update_member_info(guild_id, user_id, user.name, user.discriminator,
                       [role.name for role in user.roles], user.avatar.url, 0, 0, 1)

    await interaction.response.send_message(f"Member info for {user.name} updated.", ephemeral=True)

# Command for showing member info (owner only)
@bot.tree.command(name="member_info", description="Show a member's information.")
@commands.is_owner()
async def member_info(interaction: discord.Interaction, user: discord.User):
    guild_id = interaction.guild.id
    db_name = get_db_name(guild_id)

    conn = sqlite3.connect(db_name)
    cursor = conn.cursor()
    cursor.execute('SELECT * FROM members WHERE MEMBER_ID=?', (user.id,))
    member_data = cursor.fetchone()
    conn.close()

    if member_data:
        embed = discord.Embed(title=f"Member Info for {user.name}", color=discord.Color.green())
        embed.add_field(name="ID", value=member_data[0], inline=True)
        embed.add_field(name="Username", value=member_data[1], inline=True)
        embed.add_field(name="Discriminator", value=member_data[2], inline=True)
        embed.add_field(name="Join Date", value=member_data[3], inline=True)
        embed.add_field(name="Join Time", value=member_data[4], inline=True)
        embed.add_field(name="Roles", value=json.loads(member_data[5]), inline=True)
        embed.add_field(name="Avatar URL", value=member_data[6], inline=True)
        embed.add_field(name="Warnings", value=member_data[7], inline=True)
        embed.add_field(name="Premium Status", value=bool(member_data[8]), inline=True)
        embed.add_field(name="ToS Accepted", value=bool(member_data[9]), inline=True)
        await interaction.response.send_message(embed=embed, ephemeral=True)
    else:
        await interaction.response.send_message(f"No data found for {user.name}.", ephemeral=True)

# Event to save new member info when they join
@bot.event
async def on_member_join(member):
    guild_id = member.guild.id
    await create_table(guild_id)  # Ensure the table exists
    update_member_info(guild_id, member.id, member.name, member.discriminator,
                       [role.name for role in member.roles], member.avatar.url, 0, 0, 0)  # Default warnings, premium status, ToS

# Event to update member info when it changes
@bot.event
async def on_member_update(before, after):
    guild_id = after.guild.id
    update_member_info(guild_id, after.id, after.name, after.discriminator,
                       [role.name for role in after.roles], after.avatar.url, 0, 0, after.tos)


@bot.tree.command(name= 'website', description= 'Opens the Official Dialex Website')
async def website(interaction: discord.Interaction):

    website = f'[Here is the Website!](https://doctorwhoii.github.io/Dialex-Website/)'
    message_ = f'Hello there\n this is the official Dialex Website!\n here you can find all our servers and a link to add Dialex to your own\n **{website}**'

    embed = discord.Embed(title= f'Website', description= message_, color=discord.Color.blue())

    await interaction.response.send_message(embed=embed)

@bot.event
async def on_message(message):
    # Prevent the bot from responding to its own messages
    if message.author == bot.user:
        return

    # Check if the message contains "Vanity" or "Server"
    if "Vanity" in message.content or "Server" in message.content or "server" in message.content or "vanity" in message.content:
        await message.channel.send("https://discord.gg/UgNN2zcarf")

    # Process commands after checking the message
    await bot.process_commands(message)

@bot.tree.command(name="math", description="Evaluate a complex math expression.")
async def math(interaction: discord.Interaction, expression: str):
    try:
        # Parse the expression using sympy
        result = sp.sympify(expression)

        # Format the result
        formatted_result = sp.latex(result)

        # Send the result back
        await interaction.response.send_message(f"The result of `{expression}` is: {result} (Latex: {formatted_result})")
    except Exception as e:
        await interaction.response.send_message(f"Error evaluating expression: {e}", ephemeral=True)

# Buddy System
volunteer_role_name = 'Buddy Volunteer'

class VolunteerButton(discord.ui.Button):
    def __init__(self):
        super().__init__(style=discord.ButtonStyle.primary, label="Volunteer")

    async def callback(self, interaction: discord.Interaction):
        role = discord.utils.get(interaction.guild.roles, name=volunteer_role_name)
        if role:
            await interaction.user.add_roles(role)
            await interaction.response.send_message(f"You have been given the `{volunteer_role_name}` role. Thank you for volunteering!", ephemeral=True)
        else:
            await interaction.response.send_message(f"The role `{volunteer_role_name}` does not exist.", ephemeral=True)

class BuddyButton(discord.ui.Button):
    def __init__(self):
        super().__init__(style=discord.ButtonStyle.primary, label="Find a Buddy")

    async def callback(self, interaction: discord.Interaction):
        role = discord.utils.get(interaction.guild.roles, name=volunteer_role_name)
        if not role:
            await interaction.response.send_message(f"The role `{volunteer_role_name}` does not exist.", ephemeral=True)
            return

        new_member = interaction.user
        volunteers = [member for member in interaction.guild.members if role in member.roles and member != new_member]

        if not volunteers:
            await interaction.response.send_message("There are no available volunteers at the moment. Please try again later.", ephemeral=True)
            return

        buddy = random.choice(volunteers)
        if isinstance(interaction.channel, discord.DMChannel):
            await interaction.response.send_message("This command can only be used in a server channel.", ephemeral=True)
            return

        thread_name = f"Buddy Request: {new_member.display_name}"
        thread = await interaction.channel.create_thread(name=f"Buddy System - {new_member.display_name}")
        await thread.send(f"Hello {new_member.mention} and {buddy.mention}, welcome to the Buddy System! Feel free to get to know each other here.")

        await interaction.response.send_message(f"Private channel created with {buddy.mention}. Check the new channel for your buddy session.", ephemeral=True)

        # Set a timer to close the thread if inactive
        await asyncio.sleep(3 * 3600)
        if len(await thread.history(limit=1).flatten()) == 0:
            await thread.delete()
            await interaction.channel.send(f"The buddy channel with {new_member.mention} and {buddy.mention} was closed due to inactivity.")

# Buddy System Command
@bot.tree.command(name="buddy-system", description="Start the buddy system to make friends or volunteer.")
async def buddy_system(interaction: discord.Interaction):
    view = discord.ui.View()
    view.add_item(BuddyButton())
    view.add_item(VolunteerButton())

    embed = discord.Embed(
        title="Buddy System",
        description=(
            "New? No problem, make a friend here using the buddy system. Click the button, and you'll be put in a private channel with an active member where you can bond a friendship.\n\n"
            "Maybe you are an active member and would like to volunteer for this position? Click the blue button saying 'Volunteer'."
        ),
        color=discord.Color.blue()
    )

    await interaction.response.send_message(embed=embed, view=view, ephemeral=True)

@bot.hybrid_command(name='remove-buddy-role', description='Remove the Buddy Volunteer role from yourself')
async def remove_buddy_role(ctx: commands.Context):
    # Get the role to remove
    role = discord.utils.get(ctx.guild.roles, name=volunteer_role_name)

    # Check if the role exists
    if not role:
        await ctx.send(f"The role `{volunteer_role_name}` does not exist.")
        return

    # Check if the bot has permission to manage roles
    if not ctx.guild.me.guild_permissions.manage_roles:
        await ctx.send("I don't have permission to manage roles.")
        return

    # Check if the role is higher than the bot's top role
    if role.position >= ctx.guild.me.top_role.position:
        await ctx.send("I cannot manage this role because it is higher or equal to my highest role.")
        return

    # Check if the user has the role
    if role not in ctx.author.roles:
        await ctx.send(f"You do not have the `{volunteer_role_name}` role.")
        return

    # Remove the role from the user
    try:
        await ctx.author.remove_roles(role)
        await ctx.send(f"Successfully removed the `{volunteer_role_name}` role from yourself.")
    except Exception as e:
        await ctx.send(f"An error occurred: {e}")


# Check Message Count Command
@bot.hybrid_command(name='check-chats', description='Check message count between two dates (format: YYYY/MM/DD)')
@commands.has_permissions(manage_messages=True)
async def check_chats(ctx, user: discord.Member, start_date: str, end_date: str):
    try:
        start_date = datetime.strptime(start_date, '%Y/%m/%d')
        end_date = datetime.strptime(end_date, '%Y/%m/%d')
    except ValueError:
        await ctx.send("Date format is incorrect. Please use YYYY/MM/DD.")
        return

    if start_date > end_date:
        await ctx.send("The start date cannot be after the end date.")
        return

    message_count = 0
    after = start_date
    before = end_date

    async for message in ctx.channel.history(after=after, before=before, oldest_first=True, limit=None):
        if message.author == user:
            message_count += 1

    result = f"{user.mention} sent {message_count} messages between {start_date.strftime('%Y/%m/%d')} and {end_date.strftime('%Y/%m/%d')}."
    await ctx.send(result)



# Sample phrases and their emoji representations
emoji_phrases = {
    "The Lion King": "🦁👑",
    "Ice Cream": "🍦",
    "Finding Nemo": "🔍🐠",
    "Star Wars": "⭐️⚔️",
    "Game of Thrones": "🐉👑",
    "Harry Potter": "⚡️🧙‍♂️",
    "The Matrix": "💊🕶️",
    "SpongeBob SquarePants": "🍍🧽",
    "Jurassic Park": "🦖🏞️",
    "The Godfather": "👨‍👧‍👦🔫",
    "Frozen": "❄️👸",
    "Pirates of the Caribbean": "🏴‍☠️🌊",
    "Avatar": "🌍👽🔵",
    "The Avengers": "🦸‍♂️🦸‍🔨🛡️💎",
    "Batman": "🦇👨",
}

# Dictionary to keep track of scores
scores = {}

@bot.tree.command(name='emoji_pictionary', description='Start a game of Emoji Pictionary.')
async def emoji_pictionary(interaction: discord.Interaction, rounds: int = 3):
    for _ in range(rounds):
        phrase, emoji_sequence = random.choice(list(emoji_phrases.items()))
        await interaction.response.send_message(f"Guess the phrase represented by these emojis: {emoji_sequence}", ephemeral=False)

        def check(m):
            return m.channel == interaction.channel and m.author != bot.user

        try:
            msg = await bot.wait_for('message', check=check, timeout=30.0)
            if msg.content.lower() == phrase.lower():
                # Update the score for the user
                if msg.author not in scores:
                    scores[msg.author] = 0
                scores[msg.author] += 1
                await interaction.followup.send(f"Congratulations {msg.author.mention}, you guessed it! The phrase was: **{phrase}**")
            else:
                await interaction.followup.send(f"Sorry! The correct answer was: **{phrase}**")
        except asyncio.TimeoutError:
            await interaction.followup.send(f"Time's up! The correct answer was: **{phrase}**")

    # Display final scores
    score_message = "Final Scores:\n"
    for user, score in scores.items():
        score_message += f"{user.mention}: {score}\n"
    await interaction.followup.send(score_message)

    # Reset scores for the next game
    scores.clear()



# Token Setup
TOKEN = 'hey there baby~'
if not TOKEN:
    raise Exception("Please add your token to the environment variables.")

bot.run(TOKEN)
