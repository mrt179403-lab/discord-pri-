# discord-pri-
import discord
from discord.ext import commands
from discord import app_commands
import os
import json
import asyncio
from typing import Optional

# ──────────────────────────────────────────────
# Config & Storage
# ──────────────────────────────────────────────
TOKEN = os.environ.get("MTUwMDQ1OTczNTA0MzAxODg0Mw.GgOt48.7UFHh5nbiYQAzpMkixWpH8l1RCryRJPR6_icmM", "")
DATA_FILE = "data.json"

def load_data():
    if os.path.exists(DATA_FILE):
        with open(DATA_FILE, "r") as f:
            return json.load(f)
    return {"ping_roles": {}, "role_panels": {}}

def save_data(data):
    with open(DATA_FILE, "w") as f:
        json.dump(data, f, indent=2)

data = load_data()

# ──────────────────────────────────────────────
# Bot Setup
# ──────────────────────────────────────────────
intents = discord.Intents.default()
intents.members = True
intents.message_content = True

bot = commands.Bot(command_prefix="!", intents=intents)
tree = bot.tree

# ──────────────────────────────────────────────
# Events
# ──────────────────────────────────────────────
@bot.event
async def on_ready():
    await tree.sync()
    print(f"✅ Bot is online as {bot.user}")

@bot.event
async def on_member_update(before: discord.Member, after: discord.Member):
    """Track ping role membership changes and update counts"""
    guild_id = str(after.guild.id)
    if guild_id not in data["ping_roles"]:
        return

    before_roles = set(r.id for r in before.roles)
    after_roles = set(r.id for r in after.roles)

    added = after_roles - before_roles
    removed = before_roles - after_roles

    changed = False
    for role_id_str, role_data in data["ping_roles"][guild_id].items():
        role_id = int(role_id_str)
        if role_id in added:
            role_data["count"] = role_data.get("count", 0) + 1
            changed = True
        elif role_id in removed:
            role_data["count"] = max(0, role_data.get("count", 0) - 1)
            changed = True

    if changed:
        save_data(data)

# ──────────────────────────────────────────────
# /kick
# ──────────────────────────────────────────────
@tree.command(name="kick", description="Kick a member from the server")
@app_commands.describe(member="The member to kick", reason="Reason for kick")
@app_commands.checks.has_permissions(kick_members=True)
async def kick(interaction: discord.Interaction, member: discord.Member, reason: Optional[str] = "No reason provided"):
    if member.top_role >= interaction.user.top_role:
        await interaction.response.send_message("❌ You can't kick someone with equal or higher role.", ephemeral=True)
        return
    await member.kick(reason=reason)
    embed = discord.Embed(title="👢 Member Kicked", color=discord.Color.orange())
    embed.add_field(name="User", value=f"{member} ({member.id})", inline=False)
    embed.add_field(name="Moderator", value=str(interaction.user), inline=True)
    embed.add_field(name="Reason", value=reason, inline=True)
    await interaction.response.send_message(embed=embed)

# ──────────────────────────────────────────────
# /ban
# ──────────────────────────────────────────────
@tree.command(name="ban", description="Ban a member from the server")
@app_commands.describe(member="The member to ban", reason="Reason for ban", delete_days="Days of messages to delete (0-7)")
@app_commands.checks.has_permissions(ban_members=True)
async def ban(interaction: discord.Interaction, member: discord.Member, reason: Optional[str] = "No reason provided", delete_days: Optional[int] = 0):
    if member.top_role >= interaction.user.top_role:
        await interaction.response.send_message("❌ You can't ban someone with equal or higher role.", ephemeral=True)
        return
    await member.ban(reason=reason, delete_message_days=min(7, max(0, delete_days)))
    embed = discord.Embed(title="🔨 Member Banned", color=discord.Color.red())
    embed.add_field(name="User", value=f"{member} ({member.id})", inline=False)
    embed.add_field(name="Moderator", value=str(interaction.user), inline=True)
    embed.add_field(name="Reason", value=reason, inline=True)
    await interaction.response.send_message(embed=embed)

# ──────────────────────────────────────────────
# /mute
# ──────────────────────────────────────────────
@tree.command(name="mute", description="Timeout (mute) a member")
@app_commands.describe(member="The member to mute", duration="Duration in minutes", reason="Reason for mute")
@app_commands.checks.has_permissions(moderate_members=True)
async def mute(interaction: discord.Interaction, member: discord.Member, duration: int = 10, reason: Optional[str] = "No reason provided"):
    if member.top_role >= interaction.user.top_role:
        await interaction.response.send_message("❌ You can't mute someone with equal or higher role.", ephemeral=True)
        return
    until = discord.utils.utcnow() + discord.utils.timedelta(minutes=duration)
    await member.timeout(until, reason=reason)
    embed = discord.Embed(title="🔇 Member Muted", color=discord.Color.yellow())
    embed.add_field(name="User", value=f"{member} ({member.id})", inline=False)
    embed.add_field(name="Duration", value=f"{duration} minutes", inline=True)
    embed.add_field(name="Moderator", value=str(interaction.user), inline=True)
    embed.add_field(name="Reason", value=reason, inline=False)
    await interaction.response.send_message(embed=embed)

# ──────────────────────────────────────────────
# /panelticket
# ──────────────────────────────────────────────
@tree.command(name="panelticket", description="Send a ticket panel in this channel")
@app_commands.describe(title="Panel title", description="Panel description")
@app_commands.checks.has_permissions(manage_channels=True)
async def panelticket(interaction: discord.Interaction, title: str = "🎫 Support Tickets", description: str = "Click the button below to open a support ticket."):
    embed = discord.Embed(title=title, description=description, color=discord.Color.blue())
    embed.set_footer(text="One ticket per user")
    view = TicketView()
    await interaction.response.send_message("✅ Ticket panel sent!", ephemeral=True)
    await interaction.channel.send(embed=embed, view=view)

class TicketView(discord.ui.View):
    def __init__(self):
        super().__init__(timeout=None)

    @discord.ui.button(label="Open Ticket", style=discord.ButtonStyle.green, emoji="🎫", custom_id="open_ticket")
    async def open_ticket(self, interaction: discord.Interaction, button: discord.ui.Button):
        guild = interaction.guild
        existing = discord.utils.get(guild.channels, name=f"ticket-{interaction.user.name.lower()}")
        if existing:
            await interaction.response.send_message(f"❌ You already have an open ticket: {existing.mention}", ephemeral=True)
            return

        overwrites = {
            guild.default_role: discord.PermissionOverwrite(view_channel=False),
            interaction.user: discord.PermissionOverwrite(view_channel=True, send_messages=True),
            guild.me: discord.PermissionOverwrite(view_channel=True, send_messages=True),
        }
        # Add permissions for admins
        for role in guild.roles:
            if role.permissions.administrator:
                overwrites[role] = discord.PermissionOverwrite(view_channel=True, send_messages=True)

        channel = await guild.create_text_channel(
            f"ticket-{interaction.user.name.lower()}",
            overwrites=overwrites,
            topic=f"Ticket for {interaction.user}"
        )
        embed = discord.Embed(
            title="🎫 Ticket Opened",
            description=f"Hello {interaction.user.mention}! Support will be with you shortly.\nClick 🔒 to close this ticket.",
            color=discord.Color.green()
        )
        close_view = CloseTicketView()
        await channel.send(embed=embed, view=close_view)
        await interaction.response.send_message(f"✅ Ticket created: {channel.mention}", ephemeral=True)

class CloseTicketView(discord.ui.View):
    def __init__(self):
        super().__init__(timeout=None)

    @discord.ui.button(label="Close Ticket", style=discord.ButtonStyle.red, emoji="🔒", custom_id="close_ticket")
    async def close_ticket(self, interaction: discord.Interaction, button: discord.ui.Button):
        await interaction.response.send_message("🔒 Closing ticket in 5 seconds...")
        await asyncio.sleep(5)
        await interaction.channel.delete(reason=f"Ticket closed by {interaction.user}")

# ──────────────────────────────────────────────
# /inforole — Ping Roles Info Embed
# ──────────────────────────────────────────────
@tree.command(name="inforole", description="Show ping roles info with live member counts")
@app_commands.checks.has_permissions(manage_roles=True)
async def inforole(interaction: discord.Interaction):
    guild_id = str(interaction.guild.id)
    guild = interaction.guild

    if guild_id not in data["ping_roles"] or not data["ping_roles"][guild_id]:
        await interaction.response.send_message("❌ No ping roles configured yet. Use `/inforole_setup` to add roles.", ephemeral=True)
        return

    embed = discord.Embed(title="🔔 Ping Roles", color=discord.Color.blurple())
    lines = []
    for role_id_str, role_data in data["ping_roles"][guild_id].items():
        role = guild.get_role(int(role_id_str))
        if role:
            count = role_data.get("count", 0)
            lines.append(f"{role.mention} • **{count}** members")

    embed.description = "\n".join(lines) if lines else "No roles found."
    await interaction.response.send_message(embed=embed)

@tree.command(name="inforole_setup", description="Add a ping role to track")
@app_commands.describe(role="The role to track as a ping role")
@app_commands.checks.has_permissions(manage_roles=True)
async def inforole_setup(interaction: discord.Interaction, role: discord.Role):
    guild_id = str(interaction.guild.id)
    if guild_id not in data["ping_roles"]:
        data["ping_roles"][guild_id] = {}

    role_id_str = str(role.id)
    # Count current members
    current_count = sum(1 for m in interaction.guild.members if role in m.roles)
    data["ping_roles"][guild_id][role_id_str] = {"name": role.name, "count": current_count}
    save_data(data)
    await interaction.response.send_message(f"✅ Added **{role.name}** to ping roles with **{current_count}** current members.", ephemeral=True)

@tree.command(name="inforole_remove", description="Remove a ping role from tracking")
@app_commands.describe(role="The role to remove")
@app_commands.checks.has_permissions(manage_roles=True)
async def inforole_remove(interaction: discord.Interaction, role: discord.Role):
    guild_id = str(interaction.guild.id)
    role_id_str = str(role.id)
    if guild_id in data["ping_roles"] and role_id_str in data["ping_roles"][guild_id]:
        del data["ping_roles"][guild_id][role_id_str]
        save_data(data)
        await interaction.response.send_message(f"✅ Removed **{role.name}** from ping roles.", ephemeral=True)
    else:
        await interaction.response.send_message("❌ That role isn't being tracked.", ephemeral=True)

# ──────────────────────────────────────────────
# /managerrole — Role Manager Panel (up to 7 roles)
# ──────────────────────────────────────────────
@tree.command(name="managerrole", description="Create a role manager panel (up to 7 roles with custom buttons)")
@app_commands.describe(
    title="Panel title",
    description="Panel description",
    role1="Role 1", emoji1="Emoji for role 1", label1="Button label for role 1",
    role2="Role 2", emoji2="Emoji for role 2", label2="Button label for role 2",
    role3="Role 3", emoji3="Emoji for role 3", label3="Button label for role 3",
    role4="Role 4", emoji4="Emoji for role 4", label4="Button label for role 4",
    role5="Role 5", emoji5="Emoji for role 5", label5="Button label for role 5",
    role6="Role 6", emoji6="Emoji for role 6", label6="Button label for role 6",
    role7="Role 7", emoji7="Emoji for role 7", label7="Button label for role 7",
)
@app_commands.checks.has_permissions(manage_roles=True)
async def managerrole(
    interaction: discord.Interaction,
    title: str = "🎭 Role Manager",
    description: str = "Add the roles you want to be mentioned for.",
    role1: Optional[discord.Role] = None, emoji1: Optional[str] = None, label1: Optional[str] = None,
    role2: Optional[discord.Role] = None, emoji2: Optional[str] = None, label2: Optional[str] = None,
    role3: Optional[discord.Role] = None, emoji3: Optional[str] = None, label3: Optional[str] = None,
    role4: Optional[discord.Role] = None, emoji4: Optional[str] = None, label4: Optional[str] = None,
    role5: Optional[discord.Role] = None, emoji5: Optional[str] = None, label5: Optional[str] = None,
    role6: Optional[discord.Role] = None, emoji6: Optional[str] = None, label6: Optional[str] = None,
    role7: Optional[discord.Role] = None, emoji7: Optional[str] = None, label7: Optional[str] = None,
):
    roles_config = []
    for role, emoji, label in [
        (role1, emoji1, label1), (role2, emoji2, label2), (role3, emoji3, label3),
        (role4, emoji4, label4), (role5, emoji5, label5), (role6, emoji6, label6),
        (role7, emoji7, label7),
    ]:
        if role:
            roles_config.append({
                "role_id": role.id,
                "emoji": emoji or "🔹",
                "label": label or role.name,
            })

    if not roles_config:
        await interaction.response.send_message("❌ You must provide at least 1 role.", ephemeral=True)
        return

    # Save panel config
    guild_id = str(interaction.guild.id)
    panel_id = f"rp_{interaction.id}"
    if guild_id not in data["role_panels"]:
        data["role_panels"][guild_id] = {}
    data["role_panels"][guild_id][panel_id] = roles_config
    save_data(data)

    embed = discord.Embed(title=title, description=description, color=discord.Color.blurple())
    view = RoleManagerView(roles_config)
    await interaction.response.send_message("✅ Role panel sent!", ephemeral=True)
    await interaction.channel.send(embed=embed, view=view)

class RoleManagerView(discord.ui.View):
    def __init__(self, roles_config: list):
        super().__init__(timeout=None)
        for i, cfg in enumerate(roles_config[:7]):
            btn = RoleButton(
                role_id=cfg["role_id"],
                label=cfg["label"],
                emoji=cfg["emoji"],
                custom_id=f"rolebtn_{cfg['role_id']}_{i}"
            )
            self.add_item(btn)

class RoleButton(discord.ui.Button):
    def __init__(self, role_id: int, label: str, emoji: str, custom_id: str):
        super().__init__(
            style=discord.ButtonStyle.blurple,
            label=label,
            emoji=emoji,
            custom_id=custom_id
        )
        self.role_id = role_id

    async def callback(self, interaction: discord.Interaction):
        guild = interaction.guild
        role = guild.get_role(self.role_id)
        if not role:
            await interaction.response.send_message("❌ Role not found.", ephemeral=True)
            return

        member = interaction.user
        if role in member.roles:
            await member.remove_roles(role)
            await interaction.response.send_message(f"✅ Removed **{role.name}** from you.", ephemeral=True)
        else:
            await member.add_roles(role)
            await interaction.response.send_message(f"✅ Added **{role.name}** to you.", ephemeral=True)

# ──────────────────────────────────────────────
# Error Handlers
# ──────────────────────────────────────────────
@tree.error
async def on_app_command_error(interaction: discord.Interaction, error: app_commands.AppCommandError):
    if isinstance(error, app_commands.MissingPermissions):
        await interaction.response.send_message("❌ You don't have permission to use this command.", ephemeral=True)
    else:
        await interaction.response.send_message(f"❌ Error: {str(error)}", ephemeral=True)

# ──────────────────────────────────────────────
# Run
# ──────────────────────────────────────────────
if __name__ == "__main__":
    if not TOKEN:
        print("❌ MTUwMDQ1OTczNTA0MzAxODg0Mw.GgOt48.7UFHh5nbiYQAzpMkixWpH8l1RCryRJPR6_icmM")
    else:
        bot.run(TOKEN)
