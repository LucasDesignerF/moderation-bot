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
            muted_role = await ctx.guild.create_role(name="Muted")
            for channel in ctx.guild.channels:
                await channel.set_permissions(muted_role, speak=False, send_messages=False)

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

    # Comando de unban
    @commands.slash_command(guild_ids=[...])
    async def unban(self, ctx, user_id: int):
        """Desbane um membro do servidor."""
        user = await self.bot.fetch_user(user_id)
        await ctx.guild.unban(user)
        await ctx.respond(f"✅ {user.mention} foi desbanido.")

    # Comando de unwarn
    @commands.slash_command(guild_ids=[...])
    async def unwarn(self, ctx, member: Member, index: int):
        """Remove um aviso específico de um membro."""
        self.ensure_server_data(ctx.guild.id)
        path = f"./data/{ctx.guild.id}/moderation.json"

        with open(path, "r") as file:
            data = json.load(file)

        warns = data["warns"].get(str(member.id), [])
        if 0 <= index - 1 < len(warns):
            removed_warning = warns.pop(index - 1)
            with open(path, "w") as file:
                json.dump(data, file, indent=4)
            await ctx.respond(f"Aviso removido: {removed_warning}")
        else:
            await ctx.respond(f"Aviso não encontrado para {member.mention}.")

    # Comando de unmute
    @commands.slash_command(guild_ids=[...])
    async def unmute(self, ctx, member: Member):
        """Remove o silêncio de um membro."""
        muted_role = discord.utils.get(ctx.guild.roles, name="Muted")
        if muted_role in member.roles:
            await member.remove_roles(muted_role)
            self.muted_users.pop(member.id, None)
            await ctx.respond(f"🔈 {member.mention} foi desmutado.")
        else:
            await ctx.respond(f"{member.mention} não está silenciado.")

    # Comando de baninfo
    @commands.slash_command(guild_ids=[...])
    async def baninfo(self, ctx, user_id: int):
        """Exibe informações sobre o banimento de um usuário."""
        bans = await ctx.guild.bans()
        user_ban = discord.utils.find(lambda ban: ban.user.id == user_id, bans)
        if user_ban:
            await ctx.respond(f"Usuário: {user_ban.user}\nMotivo: {user_ban.reason}")
        else:
            await ctx.respond("Este usuário não está banido.")

    # Comando de warn
    @commands.slash_command(guild_ids=[...])
    async def warn(self, ctx, member: Member, role: discord.Role, *, reason: str):
        """Dá um aviso a um membro e atribui um cargo específico."""
        self.ensure_server_data(ctx.guild.id)
        path = f"./data/{ctx.guild.id}/moderation.json"

        with open(path, "r") as file:
            data = json.load(file)

        if str(member.id) not in data["warns"]:
            data["warns"][str(member.id)] = []
        data["warns"][str(member.id)].append(reason)

        with open(path, "w") as file:
            json.dump(data, file, indent=4)

        await member.add_roles(role, reason=f"Aviso: {reason}")
        await ctx.respond(f"{member.mention} foi avisado. Motivo: {reason}. Cargo atribuído: {role.name}")

    # Comando de warnlist
    @commands.slash_command(guild_ids=[...])
    async def warnlist(self, ctx, member: Member):
        """Lista todos os avisos de um membro."""
        path = f"./data/{ctx.guild.id}/moderation.json"
        
        with open(path, "r") as file:
            data = json.load(file)

        warns = data["warns"].get(str(member.id), [])
        if not warns:
            await ctx.respond(f"{member.mention} não tem avisos.")
        else:
            warnings_list = "\n".join(f"{idx+1}. {reason}" for idx, reason in enumerate(warns))
            await ctx.respond(f"Avisos de {member.mention}:\n{warnings_list}")

    # Comando de lock
    @commands.slash_command(guild_ids=[...])
    async def lock(self, ctx, channel: discord.TextChannel = None):
        """Bloqueia mensagens em um canal."""
        channel = channel or ctx.channel
        overwrite = channel.overwrites_for(ctx.guild.default_role)
        overwrite.send_messages = False
        await channel.set_permissions(ctx.guild.default_role, overwrite=overwrite)
        await ctx.respond(f"🔒 {channel.mention} foi bloqueado.")

    # Comando de unlock
    @commands.slash_command(guild_ids=[...])
    async def unlock(self, ctx, channel: discord.TextChannel = None):
        """Desbloqueia mensagens em um canal."""
        channel = channel or ctx.channel
        overwrite = channel.overwrites_for(ctx.guild.default_role)
        overwrite.send_messages = True
        await channel.set_permissions(ctx.guild.default_role, overwrite=overwrite)
        await ctx.respond(f"🔓 {channel.mention} foi desbloqueado.")

def setup(bot):
    bot.add_cog(Moderation(bot))

```
# Comandos
1. Ban
- Uso: /ban <membro> <motivo>
> Descrição: Bane um membro do servidor, removendo-o e impedindo-o de retornar.
> Parâmetros:
> membro: O membro que será banido.
> motivo (opcional): O motivo do banimento (padrão: "Não especificado").
2. Kick
- Uso: /kick <membro> <motivo>
> Descrição: Expulsa um membro do servidor. O usuário pode retornar ao servidor posteriormente, diferente do banimento.
> Parâmetros:
> membro: O membro que será expulso.
> motivo (opcional): O motivo da expulsão (padrão: "Não especificado").
3. Mute
- Uso: /mute <membro> <duração> <motivo>
> Descrição: Silencia um membro por um período específico, removendo a capacidade de enviar mensagens e falar.
> Parâmetros:
> membro: O membro que será silenciado.
> duração: Tempo de silêncio em minutos.
> motivo (opcional): O motivo do silêncio (padrão: "Não especificado").
4. Unmute
- Uso: /unmute <membro>
> Descrição: Remove o silêncio de um membro.
> Parâmetros:
> membro: O membro que terá o silêncio removido.
5. Unban
- Uso: /unban <ID do usuário>
> Descrição: Remove o banimento de um usuário específico, permitindo que ele retorne ao servidor.
> Parâmetros:
> ID do usuário: ID do usuário a ser desbanido.
6. Warn
- Uso: /warn <membro> <cargo> <motivo>
> Descrição: Adiciona um aviso ao membro e atribui um cargo específico, usado para destacar membros que cometeram infrações.
> Parâmetros:
> membro: O membro que receberá o aviso.
> cargo: O cargo a ser atribuído ao membro.
> motivo: Motivo do aviso.
7. Unwarn
- Uso: /unwarn <membro> <índice>
> Descrição: Remove um aviso específico do membro, usando o índice do aviso na lista.
> Parâmetros:
> membro: O membro que terá o aviso removido.
> índice: O número do aviso na lista (iniciado em 1).
8. Warnlist
- Uso: /warnlist <membro>
> Descrição: Lista todos os avisos dados a um membro.
> Parâmetros:
> membro: O membro cujos avisos serão listados.
9. Baninfo
- Uso: /baninfo <ID do usuário>
> Descrição: Exibe informações sobre o motivo do banimento de um usuário específico.
> Parâmetros:
> ID do usuário: O ID do usuário para buscar informações de banimento.
10. Lock
- Uso: /lock [canal]
> Descrição: Bloqueia mensagens em um canal, impedindo que membros enviem mensagens.
> Parâmetros:
> canal (opcional): O canal que será bloqueado. Caso omitido, o canal atual será bloqueado.
11. Unlock
- Uso: /unlock [canal]
> Descrição: Desbloqueia mensagens em um canal, permitindo que membros enviem mensagens.
> Parâmetros:
> canal (opcional): O canal que será desbloqueado. Caso omitido, o canal atual será desbloqueado.

# Observações
- Permissões Necessárias:
> Muitos desses comandos exigem permissões de moderação, como ban_members, kick_members, e manage_channels. Certifique-se de que o bot e o usuário que executa os comandos possuam as permissões necessárias.
- IDs dos Servidores:
> Em cada comando de barra, é necessário definir guild_ids para especificar os servidores em que os comandos estarão ativos.
Estrutura de Dados.
- A cog armazena avisos no arquivo moderation.json para cada servidor, usando uma estrutura em JSON para registrar os avisos associados a cada membro.

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
- Desenvolvido por **lrfortes**.  
- Produção [Code Connect](https://discord.gg/z5SkRN495p)  
- Desenvolvido para os membros do servidor [@Syntax](https://discord.gg/FPFauHfjmw)


# PERSEU ME COLOCA DE MODERADOR HAHAHA
