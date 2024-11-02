# Estrutura de Pastas e Arquivos
Vamos organizar o projeto para facilitar o uso das cogs, o armazenamento dos dados de cada servidor, e a gestão de configurações:
```bash
📁 DiscordBot/
├── 📁 cogs/           # Pasta para armazenar cada módulo (cog) do bot
│   └── moderation.py  # Cog de moderação, que será nosso primeiro módulo
├── 📁 data/           # Pasta onde serão armazenados os dados JSON dos servidores
│   └── server_id/     # Exemplo: 123456789012345678, uma pasta para cada servidor
│       └── moderation.json # Dados específicos para a cog de moderação
├── .env               # Arquivo para as credenciais, como o token do bot
├── main.py            # Arquivo principal do bot, onde ele é inicializado e as cogs são carregadas
└── requirements.txt   # Lista das dependências, como py-cord
```

# Arquivo `.env`
Esse arquivo será usado para armazenar o Token do bot de forma segura. Para ler o arquivo, você pode utilizar a biblioteca dotenv.

- Exemplo do `.env`:
```bash
DISCORD_TOKEN=seu_token_aqui
```

# Arquivo main.py
Este arquivo será o ponto de partida do bot. Ele incluirá:

- Configuração dos intents;
- Carregamento automático das cogs;
- Logs detalhados com emojis e status do bot.
- Vamos começar com a estrutura básica do main.py:

```python
import discord
from discord.ext import commands
from dotenv import load_dotenv
import os
import logging

# Carregar token do .env
load_dotenv()
TOKEN = os.getenv("DISCORD_TOKEN")

# Configurar logging detalhado com emojis
logging.basicConfig(level=logging.INFO, format="%(asctime)s - %(levelname)s - %(message)s")
logger = logging.getLogger("discord_bot")

# Ativar todos os intents
intents = discord.Intents.all()

# Inicializar o bot com comandos slash e intents
bot = commands.Bot(command_prefix="/", intents=intents)

# Função para carregar cogs automaticamente
def load_cogs():
    for filename in os.listdir('./cogs'):
        if filename.endswith('.py'):
            cog_name = f"cogs.{filename[:-3]}"
            try:
                bot.load_extension(cog_name)
                logger.info(f"✅ Cog carregada: {filename}")
            except Exception as e:
                logger.error(f"❌ Erro ao carregar {filename}: {e}")

@bot.event
async def on_ready():
    logger.info(f"✅ Bot conectado como {bot.user}")
    logger.info(f"📋 Servidores:")
    for guild in bot.guilds:
        logger.info(f"🔸 {guild.name} | Membros: {guild.member_count}")
    logger.info("⚙️ Carregando Cogs...")
    load_cogs()

# Rodar o bot
bot.run(TOKEN)
```
Este código inicial:

- Ativa todos os intents para permitir controle total.
- Carrega as cogs da pasta cogs automaticamente ao iniciar o bot.
- Exibe logs detalhados no console para ajudar a monitorar o status.

# Estrutura da Cog de Moderação (moderation.py)
A primeira cog vai ser para moderação. Coloque esse arquivo na pasta `cogs`.
```python
import discord
from discord.ext import commands, tasks
from discord import Member
from datetime import datetime, timedelta
import os
import json

class Moderation(commands.Cog):
    def __init__(self, bot):
        self.bot = bot
        self.muted_users = {}  # Armazenar usuários silenciados temporariamente

    async def cog_load(self):
        for guild in self.bot.guilds:
            self.ensure_server_data(guild.id)

    def ensure_server_data(self, guild_id):
        path = f"./data/{guild_id}/moderation.json"
        os.makedirs(os.path.dirname(path), exist_ok=True)
        if not os.path.exists(path):
            with open(path, "w") as file:
                json.dump({"warns": {}}, file)

    @commands.Cog.listener()
    async def on_ready(self):
        print("⚙️ Cog de Moderação carregada!")
        self.check_mutes.start()

    # Comando de ban
    @commands.slash_command(guild_ids=[...])
    async def ban(self, ctx, member: Member, *, reason: str = "Não especificado"):
        """Bane um membro do servidor."""
        await member.ban(reason=reason)
        await ctx.respond(f"🔨 {member.mention} foi banido. Motivo: {reason}")

    # Comando de kick
    @commands.slash_command(guild_ids=[...])
    async def kick(self, ctx, member: Member, *, reason: str = "Não especificado"):
        """Expulsa um membro do servidor."""
        await member.kick(reason=reason)
        await ctx.respond(f"👢 {member.mention} foi expulso. Motivo: {reason}")

    # Comando de mute com duração
    @commands.slash_command(guild_ids=[...])
    async def mute(self, ctx, member: Member, duration: int, *, reason: str = "Não especificado"):
        """Silencia um membro por uma duração específica (em minutos)."""
        muted_role = discord.utils.get(ctx.guild.roles, name="Muted")
        if not muted_role:
            # Cria um cargo "Muted" se não existir
            muted_role = await ctx.guild.create_role(name="Muted")
            for channel in ctx.guild.channels:
                await channel.set_permissions(muted_role, speak=False, send_messages=False)
        
        # Adiciona o cargo Muted ao membro
        await member.add_roles(muted_role, reason=reason)
        mute_end_time = datetime.utcnow() + timedelta(minutes=duration)
        self.muted_users[member.id] = mute_end_time
        await ctx.respond(f"🔇 {member.mention} foi silenciado por {duration} minutos. Motivo: {reason}")

    # Verificar periodicamente se o mute de algum usuário expirou
    @tasks.loop(minutes=1)
    async def check_mutes(self):
        current_time = datetime.utcnow()
        to_unmute = [user_id for user_id, end_time in self.muted_users.items() if end_time <= current_time]

        for user_id in to_unmute:
            guild = self.bot.get_guild(...)  # Defina o ID do servidor
            member = guild.get_member(user_id)
            muted_role = discord.utils.get(guild.roles, name="Muted")
            if member and muted_role:
                await member.remove_roles(muted_role)
            del self.muted_users[user_id]

    # Comando de warn com cargo específico
    @commands.slash_command(guild_ids=[...])
    async def warn(self, ctx, member: Member, role: discord.Role, *, reason: str):
        """Dá um aviso a um membro e atribui um cargo específico."""
        self.ensure_server_data(ctx.guild.id)
        path = f"./data/{ctx.guild.id}/moderation.json"
        
        with open(path, "r") as file:
            data = json.load(file)

        # Incrementar avisos
        if str(member.id) not in data["warns"]:
            data["warns"][str(member.id)] = []
        data["warns"][str(member.id)].append(reason)

        # Salvar no JSON
        with open(path, "w") as file:
            json.dump(data, file, indent=4)

        # Atribuir o cargo de aviso
        await member.add_roles(role, reason=f"Aviso: {reason}")
        await ctx.respond(f"{member.mention} foi avisado. Motivo: {reason}. Cargo atribuído: {role.name}")

    # Comando de warnings para listar os avisos de um membro
    @commands.slash_command(guild_ids=[...])
    async def warnings(self, ctx, member: Member):
        """Mostra todos os avisos de um membro."""
        path = f"./data/{ctx.guild.id}/moderation.json"
        
        with open(path, "r") as file:
            data = json.load(file)

        warns = data["warns"].get(str(member.id), [])
        if not warns:
            await ctx.respond(f"{member.mention} não tem avisos.")
        else:
            warnings_list = "\n".join(f"{idx+1}. {reason}" for idx, reason in enumerate(warns))
            await ctx.respond(f"Avisos de {member.mention}:\n{warnings_list}")

def setup(bot):
    bot.add_cog(Moderation(bot))

```
Neste código:

- Ban: Usa member.ban() para banir um usuário do servidor.
- Kick: Usa member.kick() para expulsar um usuário do servidor.
- Mute com duração:
> Usa discord.utils.get() para buscar ou criar um cargo Muted e configurá-lo para impedir o envio de mensagens ou falas em canais.
> Armazena a duração do mute em muted_users e verifica periodicamente se o tempo expirou usando uma tasks.loop.
- Warn com cargo:
> Armazena o motivo do aviso no arquivo JSON e atribui o cargo especificado ao membro com member.add_roles().

# Instalação das Dependências
No arquivo `requirements.txt`, adicione:
```bash
py-cord
python-dotenv
```
Para instalar as dependências, execute:
```cmd
pip install -r requirements.txt
```

# Como Rodar o Bot
Para iniciar o bot, basta executar:
```cmd
python main.py
```

Esse é um esqueleto inicial completo! Com isso você poderá iniciar seu bot de Discord em Python, para moderação do seu servidor!

# Creditos
- Desenvolvido por `lrfortes`.
- Produção Code Connect(https://discord.gg/z5SkRN495p)
- Desenvolvido para os membros do servidor @Syntax(https://discord.gg/FPFauHfjmw)

#PERSEU ME COLOCA DE MODERADOR HAHAHA
