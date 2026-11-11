
import tkinter as tk
from tkinter import messagebox
from cryptography.fernet import Fernet
import hashlib
import secrets
import json
import os
import shutil
import hmac
from datetime import datetime

# CONFIGURACAO DA JANELA
janela = tk.Tk()
janela.title("RSALERTA - Sistema de Alertas")
janela.geometry("500x550")
janela.configure(bg="#F3EFE5")

# CORES
azul = "#3F679B"
fundo = "#F3EFE5"
branco = "white"

# CRIPTOGRAFIA DOS DADOS
if os.path.exists("chave.key"):
    with open("chave.key", "rb") as arquivo:
        chave = arquivo.read()
else:
    chave = Fernet.generate_key()
    with open("chave.key", "wb") as arquivo:
        arquivo.write(chave)

criptografia = Fernet(chave)

def criptografar(dado):
    return criptografia.encrypt(dado.encode()).decode()

def descriptografar(dado):
    return criptografia.decrypt(dado.encode()).decode()

# HASH DAS SENHAS
def gerar_hash(senha):
    sal = secrets.token_hex(16)
    resultado = hashlib.pbkdf2_hmac(
        "sha256", senha.encode(), sal.encode(), 100000
    )
    return sal + "$" + resultado.hex()

def verificar_senha(senha, senha_salva):
    sal, resultado = senha_salva.split("$")
    novo_hash = hashlib.pbkdf2_hmac(
        "sha256", senha.encode(), sal.encode(), 100000
    )
    return hmac.compare_digest(novo_hash.hex(), resultado)

# ARQUIVOS DO SISTEMA
arquivo_usuarios = "usuarios.json"
arquivo_alertas = "alertas.json"
arquivo_ocorrencias = "ocorrencias.json"

def carregar_dados(arquivo):
    if os.path.exists(arquivo):
        with open(arquivo, "r", encoding="utf-8") as f:
            return json.load(f)
    return {}

usuarios = carregar_dados(arquivo_usuarios)
alertas = carregar_dados(arquivo_alertas)
ocorrencias = carregar_dados(arquivo_ocorrencias)

# CADASTRO INICIAL DO ADMINISTRADOR
if "admin" not in usuarios:
    usuarios["admin"] = {
        "senha": gerar_hash("Admin@123"),
        "nome": criptografar("Administrador"),
        "tipo": "administrador"
    }

# SALVAR DADOS
def salvar_dados():
    with open(arquivo_usuarios, "w", encoding="utf-8") as f:
        json.dump(usuarios, f, indent=4)

    with open(arquivo_alertas, "w", encoding="utf-8") as f:
        json.dump(alertas, f, indent=4)

    with open(arquivo_ocorrencias, "w", encoding="utf-8") as f:
        json.dump(ocorrencias, f, indent=4)

salvar_dados()

# BACKUP AUTOMATICO DIARIO
def fazer_backup():
    data = datetime.now().strftime("%Y-%m-%d")
    pasta = "backups"

    if not os.path.exists(pasta):
        os.makedirs(pasta)

    destino = os.path.join(pasta, data)

    if not os.path.exists(destino):
        os.makedirs(destino)

        for arquivo in [
            arquivo_usuarios,
            arquivo_alertas,
            arquivo_ocorrencias
        ]:
            if os.path.exists(arquivo):
                shutil.copy(arquivo, destino)

fazer_backup()

# LIMPAR A JANELA
def limpar_janela():
    for widget in janela.winfo_children():
        widget.destroy()

# TITULO
def criar_titulo(texto):
    titulo = tk.Label(
        janela,
        text=texto,
        bg=azul,
        fg=branco,
        font=("Arial", 16, "bold"),
        width=40,
        height=3
    )
    titulo.pack(pady=10)

# TELA INICIAL
def tela_inicial():
    limpar_janela()
    criar_titulo("RSALERTA")

    descricao = tk.Label(
        janela,
        text="Sistema de Alertas e Monitoramento",
        bg=fundo,
        font=("Arial", 12)
    )
    descricao.pack(pady=15)

    botao_entrar = tk.Button(
        janela,
        text="Entrar",
        command=tela_login,
        bg=azul,
        fg=branco,
        font=("Arial", 12),
        width=20,
        cursor="hand2"
    )
    botao_entrar.pack(pady=10)

    botao_cadastrar = tk.Button(
        janela,
        text="Cadastrar",
        command=tela_cadastro,
        bg=azul,
        fg=branco,
        font=("Arial", 12),
        width=20,
        cursor="hand2"
    )
    botao_cadastrar.pack(pady=10)

# TELA DE CADASTRO
def tela_cadastro():
    limpar_janela()
    criar_titulo("Cadastro de usuario")

    tk.Label(
        janela, text="Nome completo:", bg=fundo
    ).pack()

    nome_entrada = tk.Entry(janela, width=35)
    nome_entrada.pack(pady=5)

    tk.Label(
        janela, text="Nome de usuario:", bg=fundo
    ).pack()

    usuario_entrada = tk.Entry(janela, width=35)
    usuario_entrada.pack(pady=5)

    tk.Label(
        janela, text="Senha:", bg=fundo
    ).pack()

    senha_entrada = tk.Entry(janela, width=35, show="*")
    senha_entrada.pack(pady=5)

    def fazer_cadastro():
        nome = nome_entrada.get().strip()
        usuario = usuario_entrada.get().strip()
        senha = senha_entrada.get()

        if nome == "" or usuario == "" or senha == "":
            messagebox.showwarning(
                "Aviso", "Preencha todos os campos!"
            )
            return

        if usuario in usuarios:
            messagebox.showwarning(
                "Aviso", "Este usuario ja existe!"
            )
            return

        if len(senha) < 6:
            messagebox.showwarning(
                "Aviso",
                "A senha deve ter pelo menos 6 caracteres!"
            )
            return

        usuarios[usuario] = {
            "nome": criptografar(nome),
            "senha": gerar_hash(senha),
            "tipo": "populacao"
        }

        salvar_dados()

        messagebox.showinfo(
            "Sucesso", "Cadastro realizado com sucesso!"
        )

        tela_login()

    tk.Button(
        janela,
        text="Criar conta",
        command=fazer_cadastro,
        bg=azul,
        fg=branco,
        font=("Arial", 12)
    ).pack(pady=15)

    tk.Button(
        janela,
        text="Voltar",
        command=tela_inicial
    ).pack()

# TELA DE LOGIN
def tela_login():
    limpar_janela()
    criar_titulo("Login")

    tk.Label(
        janela, text="Nome de usuario:", bg=fundo
    ).pack()

    usuario_entrada = tk.Entry(janela, width=35)
    usuario_entrada.pack(pady=8)

    tk.Label(
        janela, text="Senha:", bg=fundo
    ).pack()

    senha_entrada = tk.Entry(
        janela, width=35, show="*"
    )
    senha_entrada.pack(pady=8)

    def fazer_login():
        usuario = usuario_entrada.get().strip()
        senha = senha_entrada.get()

        if usuario == "" or senha == "":
            messagebox.showwarning(
                "Aviso", "Preencha todos os campos!"
            )
            return

        if usuario not in usuarios:
            messagebox.showerror(
                "Erro", "Usuario nao encontrado!"
            )
            return

        if verificar_senha(
            senha, usuarios[usuario]["senha"]
        ):
            messagebox.showinfo(
                "Sucesso", "Login realizado com sucesso!"
            )

            if usuarios[usuario]["tipo"] == "administrador":
                tela_administrador()
            else:
                tela_populacao(usuario)

        else:
            messagebox.showerror(
                "Erro", "Senha incorreta!"
            )

    tk.Button(
        janela,
        text="Entrar",
        command=fazer_login,
        bg=azul,
        fg=branco,
        font=("Arial", 12)
    ).pack(pady=15)

    tk.Button(
        janela,
        text="Voltar",
        command=tela_inicial
    ).pack()

# TELA DA POPULACAO
def tela_populacao(usuario):
    limpar_janela()
    criar_titulo("Area da Populacao")

    tk.Label(
        janela,
        text="Bem-vindo ao RSALERTA!",
        bg=fundo,
        font=("Arial", 13, "bold")
    ).pack(pady=10)

    tk.Button(
        janela,
        text="Visualizar alertas",
        command=lambda: visualizar_alertas(usuario),
        bg=azul,
        fg=branco,
        width=25
    ).pack(pady=10)

    tk.Button(
        janela,
        text="Registrar ocorrencia",
        command=lambda: registrar_ocorrencia(usuario),
        bg=azul,
        fg=branco,
        width=25
    ).pack(pady=10)

    tk.Button(
        janela,
        text="Orientacoes de prevencao",
        command=lambda: mostrar_orientacoes(usuario),
        bg=azul,
        fg=branco,
        width=25
    ).pack(pady=10)

    tk.Button(
        janela,
        text="Sair",
        command=tela_inicial
    ).pack(pady=15)

# VISUALIZAR ALERTAS
def visualizar_alertas(usuario):
    limpar_janela()
    criar_titulo("Alertas e Informacoes")

    if len(alertas) == 0:
        tk.Label(
            janela,
            text="Nenhum alerta cadastrado.",
            bg=fundo
        ).pack(pady=20)

    for codigo, dados in alertas.items():
        texto = (
            "ALERTA: " + descriptografar(dados["titulo"]) +
            "\n" + descriptografar(dados["descricao"]) +
            "\nData: " + dados["data"]
        )

        tk.Label(
            janela,
            text=texto,
            bg=fundo,
            wraplength=420,
            justify="left"
        ).pack(pady=10)

    tk.Button(
        janela,
        text="Voltar ao menu",
        command=lambda: tela_populacao(usuario)
    ).pack(pady=10)

# REGISTRAR OCORRENCIA
def registrar_ocorrencia(usuario):
    limpar_janela()
    criar_titulo("Registrar ocorrencia")

    tk.Label(
        janela, text="Tipo de ocorrencia:", bg=fundo
    ).pack()

    tipo_entrada = tk.Entry(janela, width=35)
    tipo_entrada.pack(pady=5)

    tk.Label(
        janela, text="Cidade e regiao:", bg=fundo
    ).pack()

    local_entrada = tk.Entry(janela, width=35)
    local_entrada.pack(pady=5)

    tk.Label(
        janela, text="Descricao da ocorrencia:", bg=fundo
    ).pack()

    descricao_entrada = tk.Entry(janela, width=35)
    descricao_entrada.pack(pady=5)

    def salvar_ocorrencia():
        tipo = tipo_entrada.get().strip()
        local = local_entrada.get().strip()
        descricao = descricao_entrada.get().strip()

        if tipo == "" or local == "" or descricao == "":
            messagebox.showwarning(
                "Aviso", "Preencha todos os campos!"
            )
            return

        codigo = str(len(ocorrencias) + 1)

        ocorrencias[codigo] = {
            "usuario": usuario,
            "tipo": criptografar(tipo),
            "local": criptografar(local),
            "descricao": criptografar(descricao),
            "data": datetime.now().strftime(
                "%d/%m/%Y %H:%M"
            ),
            "status": "Pendente"
        }

        salvar_dados()

        messagebox.showinfo(
            "Sucesso",
            "Ocorrencia registrada com sucesso!"
        )

        # RETORNA AO MENU DA POPULACAO
        tela_populacao(usuario)

    tk.Button(
        janela,
        text="Enviar ocorrencia",
        command=salvar_ocorrencia,
        bg=azul,
        fg=branco
    ).pack(pady=15)

    tk.Button(
        janela,
        text="Voltar ao menu",
        command=lambda: tela_populacao(usuario)
    ).pack()

# ORIENTACOES
def mostrar_orientacoes(usuario):
    limpar_janela()
    criar_titulo("Orientacoes de Prevencao")

    orientacoes = (
        "1. Evite atravessar areas alagadas.\n\n"
        "2. Procure locais seguros e elevados.\n\n"
        "3. Desligue a energia eletrica se houver risco.\n\n"
        "4. Acompanhe os alertas das autoridades.\n\n"
        "5. Em emergencias, procure os servicos de defesa civil."
    )

    tk.Label(
        janela,
        text=orientacoes,
        bg=fundo,
        font=("Arial", 11),
        justify="left",
        wraplength=400
    ).pack(pady=20)

    tk.Button(
        janela,
        text="Voltar ao menu",
        command=lambda: tela_populacao(usuario)
    ).pack()

# TELA DO ADMINISTRADOR
def tela_administrador():
    limpar_janela()
    criar_titulo("Area do Administrador")

    tk.Label(
        janela,
        text="Bem-vindo, Administrador!",
        bg=fundo,
        font=("Arial", 13, "bold")
    ).pack(pady=10)

    tk.Button(
        janela,
        text="Publicar alerta",
        command=publicar_alerta,
        bg=azul,
        fg=branco,
        width=25
    ).pack(pady=15)

    tk.Button(
        janela,
        text="Consultar ocorrencias",
        command=consultar_ocorrencias,
        bg=azul,
        fg=branco,
        width=25
    ).pack(pady=15)

    tk.Button(
        janela,
        text="Sair",
        command=tela_inicial
    ).pack(pady=15)

# PUBLICAR ALERTA
def publicar_alerta():
    limpar_janela()
    criar_titulo("Publicar alerta")

    tk.Label(
        janela, text="Titulo do alerta:", bg=fundo
    ).pack()

    titulo_entrada = tk.Entry(janela, width=35)
    titulo_entrada.pack(pady=5)

    tk.Label(
        janela, text="Descricao e orientacoes:", bg=fundo
    ).pack()

    descricao_entrada = tk.Entry(janela, width=35)
    descricao_entrada.pack(pady=5)

    def salvar_alerta():
        titulo = titulo_entrada.get().strip()
        descricao = descricao_entrada.get().strip()

        if titulo == "" or descricao == "":
            messagebox.showwarning(
                "Aviso", "Preencha todos os campos!"
            )
            return

        codigo = str(len(alertas) + 1)

        alertas[codigo] = {
            "titulo": criptografar(titulo),
            "descricao": criptografar(descricao),
            "data": datetime.now().strftime(
                "%d/%m/%Y %H:%M"
            )
        }

        salvar_dados()

        messagebox.showinfo(
            "Sucesso",
            "Alerta publicado com sucesso!"
        )

        # RETORNA AO MENU DO ADMINISTRADOR
        tela_administrador()

    tk.Button(
        janela,
        text="Publicar",
        command=salvar_alerta,
        bg=azul,
        fg=branco
    ).pack(pady=15)

    tk.Button(
        janela,
        text="Voltar ao menu",
        command=tela_administrador
    ).pack()

# CONSULTAR OCORRENCIAS
def consultar_ocorrencias():
    limpar_janela()
    criar_titulo("Ocorrencias registradas")

    if len(ocorrencias) == 0:
        tk.Label(
            janela,
            text="Nenhuma ocorrencia registrada.",
            bg=fundo
        ).pack(pady=20)

    for codigo, dados in ocorrencias.items():
        texto = (
            "Ocorrencia: " + codigo +
            "\nTipo: " + descriptografar(dados["tipo"]) +
            "\nLocal: " + descriptografar(dados["local"]) +
            "\nDescricao: " + descriptografar(dados["descricao"]) +
            "\nData: " + dados["data"] +
            "\nStatus: " + dados["status"]
        )

        tk.Label(
            janela,
            text=texto,
            bg=fundo,
            justify="left",
            wraplength=420
        ).pack(pady=10)

    tk.Button(
        janela,
        text="Voltar ao menu",
        command=tela_administrador
    ).pack(pady=10)

# INICIAR SISTEMA
tela_inicial()
janela.mainloop()
