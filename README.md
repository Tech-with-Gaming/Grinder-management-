# Grinder-management-

# import discord, discord.ext.commands, and datetime modules
import discord
from discord.ext import commands, tasks
import datetime

# create a bot instance
bot = commands.Bot(command_prefix="!")

# create a global list to store the reminders
# each reminder is a dictionary with the following keys: author, user, amount, category, date
# you can use a database instead of a list for better performance and persistence
reminders = []

# gadd command: adds a user to the grinder system with the specified amount, category, and time
# format: !gadd <user> <amount> <category> <time>
# example: !gadd @John 100 1 2d
# category: 1 or 2, corresponding to different roles
# time: in days, the bot will remind the author and the user after the specified time
@bot.command()
@commands.has_permissions(administrator=True) # make the command admin only
async def gadd(ctx, user: discord.Member, amount: int, category: int, time: str):
  # check if the category is valid (1 or 2)
  if category not in [1, 2]:
    await ctx.send("Invalid category. Please enter 1 or 2.")
    return
  # check if the time is valid (in days)
  try:
    days = int(time[:-1]) # remove the last character (d) and convert to int
  except ValueError:
    await ctx.send("Invalid time. Please enter a number followed by d (for days).")
    return
  # get the role corresponding to the category
  if category == 1:
    role = ctx.guild.get_role(123456789) # replace with your role ID for category 1
  else:
    role = ctx.guild.get_role(987654321) # replace with your role ID for category 2
  # add the role to the user
  await user.add_roles(role)
  # create an embed message to confirm the addition
  embed = discord.Embed(title="Grinder Added", description=f"{user.mention} has been added to the grinder system with {amount} points in category {category}.", color=discord.Color.green())
  embed.set_footer(text=f"The bot will remind you and {user.name} in {time}.")
  await ctx.send(embed=embed)
  # calculate the reminder date
  reminder_date = datetime.datetime.now() + datetime.timedelta(days=days)
  # store the reminder information in a dictionary (you can use a database instead)
  reminder = {"author": ctx.author.id, "user": user.id, "amount": amount, "category": category, "date": reminder_date}
  # append the reminder to a list (you can use a database instead)
  reminders.append(reminder)

# gremove command: removes a user from the grinder system and the corresponding role
# format: !gremove <user>
# example: !gremove @John
@bot.command()
@commands.has_permissions(administrator=True) # make the command admin only
async def gremove(ctx, user: discord.Member):
  # check if the user has a role in the grinder system
  role1 = ctx.guild.get_role(123456789) # replace with your role ID for category 1
  role2 = ctx.guild.get_role(987654321) # replace with your role ID for category 2
  if role1 not in user.roles and role2 not in user.roles:
    await ctx.send(f"{user.mention} is not in the grinder system.")
    return
  # remove the role from the user
  if role1 in user.roles:
    await user.remove_roles(role1)
  else:
    await user.remove_roles(role2)
  # create an embed message to confirm the removal
  embed = discord.Embed(title="Grinder Removed", description=f"{user.mention} has been removed from the grinder system.", color=discord.Color.red())
  await ctx.send(embed=embed)
  # delete the reminder for the user (if any)
  for reminder in reminders: # loop through the list of reminders
    if reminder["user"] == user.id: # check if the user ID matches
      reminders.remove(reminder) # remove the reminder from the list
      break # exit the loop

# gcheck command: checks the status of a user in the grinder system
# format: !gcheck <user>
# example: !gcheck @John
@bot.command()
@commands.has_permissions(administrator=True) # make the command admin only
async def gcheck(ctx, user: discord.Member):
  # check if the user has a role in the grinder system
  role1 = ctx.guild.get_role(123456789) # replace with your role ID for category 1
  role2 = ctx.guild.get_role(987654321) # replace with your role ID for category 2
  if role1 not in user.roles and role2 not in user.roles:
    await ctx.send(f"{user.mention} is not in the grinder system.")
    return
  # get the category and the reminder date for the user
  category = None
  date = None
  for reminder in reminders: # loop through the list of reminders
    if reminder["user"] == user.id: # check if the user ID matches
      category = reminder["category"] # get the category
      date = reminder["date"] # get the reminder date
      break # exit the loop
  # create an embed message to show the status
  embed = discord.Embed(title="Grinder Status", description=f"{user.mention} is in the grinder system with category {category}.", color=discord.Color.blue())
  embed.set_footer(text=f"The bot will remind you and {user.name} on {date.strftime('%Y-%m-%d')}.")
  await ctx.send(embed=embed)

# create a background task that runs every hour
@tasks.loop(hours=1)
async def send_reminders():
  # get the current date and time
  now = datetime.datetime.now()
  # loop through the list of reminders
  for reminder in reminders:
    # check if the reminder date is equal or past the current date
    if reminder["date"] <= now:
      # get the author and the user objects from their IDs
      author = bot.get_user(reminder["author"])
      user = bot.get_user(reminder["user"])
      # create an embed message to remind them
      embed = discord.Embed(title="Grinder Reminder", description=f"{user.mention}, your grinder with {reminder['amount']} points in category {reminder['category']} has ended.", color=discord.Color.orange())
      # send the embed message to both the author and the user as direct messages
      await author.send(embed=embed)
      await user.send(embed=embed)
      # remove the reminder from the list
      reminders.remove(reminder)

# start the background task
send_reminders.start()

# run the bot with your token
bot.run("your-token-here")
