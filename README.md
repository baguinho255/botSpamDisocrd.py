# botSpamDisocrd.py
import discord
from discord.ext import commands
import re
from datetime import datetime, timedelta
import asyncio
import logging

# Configuração inicial
logging.basicConfig(level=logging.INFO)
intents = discord.Intents.default()
intents.messages = True
intents.message_content = True

bot = commands.Bot(command_prefix='!', intents=intents)

# Configurações do bot
SPAM_THRESHOLD = 3  # Número de mensagens repetidas para considerar spam
SPAM_TIME_WINDOW = 30  # Segundos para verificar mensagens repetidas
LOG_CHANNEL_NAME = "mod-logs"  # Canal para logs de moderação
ADMIN_ROLE_NAME = "Moderador"  # Nome do cargo de moderador
LINK_REGEX = re.compile(r'https?://\S+')  # Regex para detectar links

# Dicionário para rastrear mensagens de usuários
user_message_cache = {}

@bot.event
async def on_ready():
    print(f'Bot conectado como {bot.user.name}')
    await bot.change_presence(activity=discord.Activity(
        type=discord.ActivityType.watching,
        name="por spammers"
    ))

@bot.event
async def on_message(message):
    # Ignorar mensagens de bots e mensagens sem conteúdo
    if message.author.bot or not message.content:
        return

    # Verificar se a mensagem é potencialmente spam
    if await detect_spam(message):
        # Aplicar punição
        await punish_user(message)
        # Notificar moderadores
        await log_spam_incident(message)
        # Não processar comandos se for spam
        return

    await bot.process_commands(message)

async def detect_spam(message):
    user_id = message.author.id
    current_time = datetime.utcnow()
    message_content = message.content.lower().strip()
    
    # Inicializar cache para o usuário se não existir
    if user_id not in user_message_cache:
        user_message_cache[user_id] = {
            'messages': [],
            'last_warning': None
        }
    
    # Adicionar mensagem atual ao cache
    user_message_cache[user_id]['messages'].append({
        'content': message_content,
        'time': current_time,
        'channel': message.channel.name
    })
    
    # Filtrar mensagens recentes (dentro da janela de tempo)
    recent_messages = [
        msg for msg in user_message_cache[user_id]['messages']
        if current_time - msg['time'] < timedelta(seconds=SPAM_TIME_WINDOW)
    ]
    
    # Atualizar cache apenas com mensagens recentes
    user_message_cache[user_id]['messages'] = recent_messages
    
    # Verificar se há mensagens repetidas suficientes para caracterizar spam
    if len(recent_messages) >= SPAM_THRESHOLD:
        # Verificar se pelo menos SPAM_THRESHOLD mensagens são iguais
        message_counts = {}
        for msg in recent_messages:
            message_counts[msg['content']] = message_counts.get(msg['content'], 0) + 1
        
        # Se alguma mensagem foi repetida o suficiente
        if any(count >= SPAM_THRESHOLD for count in message_counts.values()):
            return True
    
    # Verificar por links suspeitos (opcional)
    if LINK_REGEX.search(message_content):
        # Contar mensagens com links recentes
        link_messages = [msg for msg in recent_messages if LINK_REGEX.search(msg['content'])]
        if len(link_messages) >= 2:  # Limite mais baixo para links
            return True
    
    return False

async def punish_user(message):
    try:
        # Aplicar timeout de 7 dias
        duration = timedelta(days=7)
        await message.author.timeout(duration, reason="Detecção automática de spam")
        
        # Enviar DM para o usuário
        try:
            embed = discord.Embed(
                title="⛔ Aviso de Moderação",
                description="Você foi silenciado automaticamente por 7 dias por possível envio de spam.",
                color=discord.Color.red()
            )
            embed.add_field(
                name="Mensagem considerada spam:",
                value=f"```{message.content[:1000]}```",
                inline=False
            )
            embed.add_field(
                name="Se você acredita que isso foi um erro:",
                value="Por favor, contate um moderador.",
                inline=False
            )
            await message.author.send(embed=embed)
        except discord.Forbidden:
            pass  # Usuário bloqueou DMs
        
        # Deletar a mensagem de spam
        try:
            await message.delete()
        except discord.Forbidden:
            pass  # Sem permissão para deletar
        
    except discord.Forbidden:
        print(f"Sem permissão para punir {message.author.name}")
    except Exception as e:
        print(f"Erro ao punir usuário: {e}")

async def log_spam_incident(message):
    # Encontrar canal de logs
    log_channel = discord.utils.get(message.guild.text_channels, name=LOG_CHANNEL_NAME)
    if not log_channel:
        print("Canal de logs não encontrado!")
        return
    
    # Criar embed de log
    embed = discord.Embed(
        title="🚨 Detecção de Spam",
        color=discord.Color.orange(),
        timestamp=datetime.utcnow()
    )
    embed.add_field(name="Usuário", value=f"{message.author.mention} ({message.author.id})", inline=False)
    embed.add_field(name="Canal", value=f"#{message.channel.name}", inline=False)
    embed.add_field(name="Mensagem", value=f"```{message.content[:1000]}```", inline=False)
    embed.add_field(name="Ação", value="Timeout de 7 dias aplicado automaticamente", inline=False)
    embed.set_footer(text="Revise esta ação abaixo")
    
    # Enviar embed com botões de ação
    view = discord.ui.View()
    
    # Botão para remover punição
    async def pardon_callback(interaction):
        if not any(role.name == ADMIN_ROLE_NAME for role in interaction.user.roles):
            await interaction.response.send_message("Você não tem permissão para isso!", ephemeral=True)
            return
        
        try:
            await message.author.timeout(None, reason=f"Punição removida por {interaction.user.name}")
            embed = discord.Embed(
                title="✅ Punição Removida",
                description=f"{message.author.mention} foi perdoado por {interaction.user.mention}",
                color=discord.Color.green()
            )
            await interaction.response.edit_message(embed=embed, view=None)
        except Exception as e:
            await interaction.response.send_message(f"Erro ao remover punição: {e}", ephemeral=True)
    
    pardon_button = discord.ui.Button(label="Remover Punição", style=discord.ButtonStyle.green)
    pardon_button.callback = pardon_callback
    view.add_item(pardon_button)
    
    # Botão para confirmar punição
    async def confirm_callback(interaction):
        if not any(role.name == ADMIN_ROLE_NAME for role in interaction.user.roles):
            await interaction.response.send_message("Você não tem permissão para isso!", ephemeral=True)
            return
        
        embed = discord.Embed(
            title="✅ Punição Confirmada",
            description=f"{message.author.mention} permanecerá punido (decisão de {interaction.user.mention})",
            color=discord.Color.red()
        )
        await interaction.response.edit_message(embed=embed, view=None)
    
    confirm_button = discord.ui.Button(label="Confirmar Punição", style=discord.ButtonStyle.red)
    confirm_button.callback = confirm_callback
    view.add_item(confirm_button)
    
    # Enviar mensagem de log
    await log_channel.send(
        content=f"📢 {ADMIN_ROLE_NAME}s: Nova detecção de spam!",
        embed=embed,
        view=view
    )

# Comando para moderadores ajustarem configurações
@bot.command(name='antispam')
@commands.has_role(ADMIN_ROLE_NAME)
async def antispam_config(ctx, setting: str = None, value: int = None):
    """Configurações do sistema anti-spam"""
    global SPAM_THRESHOLD, SPAM_TIME_WINDOW
    
    if not setting:
        embed = discord.Embed(
            title="⚙️ Configurações Atuais do Anti-Spam",
            color=discord.Color.blue()
        )
        embed.add_field(name="Limite de mensagens", value=f"`{SPAM_THRESHOLD}` mensagens iguais", inline=False)
        embed.add_field(name="Janela de tempo", value=f"`{SPAM_TIME_WINDOW}` segundos", inline=False)
        embed.set_footer(text="Use !antispam [setting] [value] para alterar")
        await ctx.send(embed=embed)
        return
    
    if setting.lower() == "threshold" and value:
        SPAM_THRESHOLD = value
        await ctx.send(f"✅ Limite de mensagens alterado para `{value}`")
    elif setting.lower() == "window" and value:
        SPAM_TIME_WINDOW = value
        await ctx.send(f"✅ Janela de tempo alterada para `{value}` segundos")
    else:
        await ctx.send("❌ Uso incorreto. Use `!antispam threshold [valor]` ou `!antispam window [valor]`")

# Comando para limpar o cache manualmente
@bot.command(name='clearcache')
@commands.has_role(ADMIN_ROLE_NAME)
async def clear_cache(ctx):
    """Limpa o cache de mensagens do anti-spam"""
    global user_message_cache
    user_message_cache = {}
    await ctx.send("✅ Cache de mensagens limpo!")


# Rodar bot com token
bot.run("Tonken bot")
