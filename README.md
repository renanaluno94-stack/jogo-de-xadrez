# jogo-de-xadrez
um jogo de xadrez de 600 de elo desenvolvido para apresentaçao da expotec 2026 da Etec alberto santos dumont
import tkinter as tk
from tkinter import messagebox
import random

# ============================================================
# XADREZ VS BOT ~600 ELO
# Feito com Python + Tkinter, sem bibliotecas externas.
# ============================================================

TAMANHO = 80
CORES = ["#F0D9B5", "#B58863"]

VALOR = {
    "P": 100, "N": 320, "B": 330, "R": 500, "Q": 900, "K": 20000
}

PECAS = {
    "wP": "♙", "wN": "♘", "wB": "♗", "wR": "♖", "wQ": "♕", "wK": "♔",
    "bP": "♟", "bN": "♞", "bB": "♝", "bR": "♜", "bQ": "♛", "bK": "♚"
}


class Xadrez:
    def __init__(self, root):
        self.root = root
        self.root.title("Xadrez vs Bot - ~600 Elo")
        self.root.resizable(False, False)

        self.canvas = tk.Canvas(
            root, width=8 * TAMANHO, height=8 * TAMANHO,
            highlightthickness=0
        )
        self.canvas.pack()

        painel = tk.Frame(root)
        painel.pack(fill="x")

        self.status = tk.Label(
            painel, text="Sua vez (brancas)", font=("Arial", 14, "bold")
        )
        self.status.pack(pady=6)

        botoes = tk.Frame(painel)
        botoes.pack(pady=(0, 8))

        tk.Button(
            botoes, text="Novo jogo", font=("Arial", 11),
            command=self.novo_jogo
        ).pack(side="left", padx=5)

        tk.Button(
            botoes, text="Sair", font=("Arial", 11),
            command=root.destroy
        ).pack(side="left", padx=5)

        self.novo_jogo()

    def novo_jogo(self):
        self.tabuleiro = [
            list("rnbqkbnr"),
            list("pppppppp"),
            list("........"),
            list("........"),
            list("........"),
            list("........"),
            list("PPPPPPPP"),
            list("RNBQKBNR")
        ]

        self.turno = "w"
        self.selecionada = None
        self.fim = False
        self.roque = {
            "wK": True, "bK": True,
            "wRk": True, "wRq": True,
            "bRk": True, "bRq": True
        }

        self.desenhar()
        self.status.config(text="Sua vez (brancas)")

    def cor(self, peca):
        if peca == ".":
            return None
        return "w" if peca.isupper() else "b"

    def tipo(self, peca):
        return peca.upper()

    def dentro(self, l, c):
        return 0 <= l < 8 and 0 <= c < 8

    def desenhar(self):
        self.canvas.delete("all")

        for l in range(8):
            for c in range(8):
                x1 = c * TAMANHO
                y1 = l * TAMANHO
                x2 = x1 + TAMANHO
                y2 = y1 + TAMANHO

                self.canvas.create_rectangle(
                    x1, y1, x2, y2,
                    fill=CORES[(l + c) % 2],
                    outline=""
                )

                if self.selecionada == (l, c):
                    self.canvas.create_rectangle(
                        x1 + 3, y1 + 3, x2 - 3, y2 - 3,
                        outline="#00AA00", width=5
                    )

                peca = self.tabuleiro[l][c]
                if peca != ".":
                    self.canvas.create_text(
                        x1 + TAMANHO / 2,
                        y1 + TAMANHO / 2,
                        text=PECAS[self.cor(peca) + self.tipo(peca)],
                        font=("DejaVu Sans", 52)
                    )

        # Coordenadas
        for c in range(8):
            self.canvas.create_text(
                c * TAMANHO + 5, 8 * TAMANHO - 8,
                text=chr(97 + c), anchor="sw",
                font=("Arial", 9, "bold")
            )

        for l in range(8):
            self.canvas.create_text(
                4, l * TAMANHO + 5,
                text=str(8 - l), anchor="nw",
                font=("Arial", 9, "bold")
            )

        self.canvas.bind("<Button-1>", self.clique)

    def clique(self, event):
        if self.fim or self.turno != "w":
            return

        c = event.x // TAMANHO
        l = event.y // TAMANHO

        if not self.dentro(l, c):
            return

        peca = self.tabuleiro[l][c]

        if self.selecionada is None:
            if peca != "." and self.cor(peca) == "w":
                self.selecionada = (l, c)
                self.desenhar()
            return

        origem = self.selecionada
        destino = (l, c)

        if destino == origem:
            self.selecionada = None
            self.desenhar()
            return

        movimentos = self.movimentos_legais(origem[0], origem[1], self.tabuleiro)

        if destino in movimentos:
            self.fazer_jogada(origem, destino)
            self.selecionada = None
            self.desenhar()

            if not self.verificar_fim():
                self.turno = "b"
                self.status.config(text="Bot pensando...")
                self.root.after(350, self.jogada_bot)
        else:
            if peca != "." and self.cor(peca) == "w":
                self.selecionada = destino
                self.desenhar()
            else:
                self.selecionada = None
                self.desenhar()

    # --------------------------------------------------------
    # Movimentos
    # --------------------------------------------------------

    def pseudo_movimentos(self, l, c, tab):
        p = tab[l][c]
        if p == ".":
            return []

        cor = self.cor(p)
        t = self.tipo(p)
        movs = []

        if t == "P":
            direcao = -1 if cor == "w" else 1
            inicio = 6 if cor == "w" else 1

            nl = l + direcao
            if self.dentro(nl, c) and tab[nl][c] == ".":
                movs.append((nl, c))

                nl2 = l + 2 * direcao
                if l == inicio and tab[nl2][c] == ".":
                    movs.append((nl2, c))

            for dc in (-1, 1):
                nc = c + dc
                nl = l + direcao
                if self.dentro(nl, nc):
                    alvo = tab[nl][nc]
                    if alvo != "." and self.cor(alvo) != cor:
                        movs.append((nl, nc))

        elif t == "N":
            for dl, dc in [
                (-2, -1), (-2, 1), (-1, -2), (-1, 2),
                (1, -2), (1, 2), (2, -1), (2, 1)
            ]:
                nl, nc = l + dl, c + dc
                if self.dentro(nl, nc):
                    if tab[nl][nc] == "." or self.cor(tab[nl][nc]) != cor:
                        movs.append((nl, nc))

        elif t in ("B", "R", "Q"):
            direcoes = []
            if t in ("B", "Q"):
                direcoes += [(-1, -1), (-1, 1), (1, -1), (1, 1)]
            if t in ("R", "Q"):
                direcoes += [(-1, 0), (1, 0), (0, -1), (0, 1)]

            for dl, dc in direcoes:
                nl, nc = l + dl, c + dc
                while self.dentro(nl, nc):
                    if tab[nl][nc] == ".":
                        movs.append((nl, nc))
                    else:
                        if self.cor(tab[nl][nc]) != cor:
                            movs.append((nl, nc))
                        break
                    nl += dl
                    nc += dc

        elif t == "K":
            for dl in (-1, 0, 1):
                for dc in (-1, 0, 1):
                    if dl == 0 and dc == 0:
                        continue
                    nl, nc = l + dl, c + dc
                    if self.dentro(nl, nc):
                        if tab[nl][nc] == "." or self.cor(tab[nl][nc]) != cor:
                            movs.append((nl, nc))

            # Roque
            if self.pode_rocar(cor, lado="k", tab=tab):
                movs.append((l, c + 2))
            if self.pode_rocar(cor, lado="q", tab=tab):
                movs.append((l, c - 2))

        return movs

    def localizar_rei(self, cor, tab):
        rei = "K" if cor == "w" else "k"
        for l in range(8):
            for c in range(8):
                if tab[l][c] == rei:
                    return (l, c)
        return None

    def casa_atacada(self, l, c, por_cor, tab):
        for ol in range(8):
            for oc in range(8):
                p = tab[ol][oc]
                if p != "." and self.cor(p) == por_cor:
                    if (l, c) in self.pseudo_sem_roque(ol, oc, tab):
                        return True
        return False

    def pseudo_sem_roque(self, l, c, tab):
        p = tab[l][c]
        if p == ".":
            return []

        # Para verificar ataques, reis não precisam de roque.
        if self.tipo(p) != "K":
            return self.pseudo_movimentos(l, c, tab)

        cor = self.cor(p)
        movs = []
        for dl in (-1, 0, 1):
            for dc in (-1, 0, 1):
                if dl == 0 and dc == 0:
                    continue
                nl, nc = l + dl, c + dc
                if self.dentro(nl, nc):
                    if tab[nl][nc] == "." or self.cor(tab[nl][nc]) != cor:
                        movs.append((nl, nc))
        return movs

    def em_xeque(self, cor, tab):
        rei = self.localizar_rei(cor, tab)
        if rei is None:
            return True
        adversaria = "b" if cor == "w" else "w"
        return self.casa_atacada(rei[0], rei[1], adversaria, tab)

    def movimentos_legais(self, l, c, tab):
        p = tab[l][c]
        if p == ".":
            return []

        cor = self.cor(p)
        legais = []

        for destino in self.pseudo_movimentos(l, c, tab):
            novo = self.copiar_tab(tab)
            self.aplicar_movimento_simples(novo, (l, c), destino)

            # Não permite capturar o rei diretamente.
            capturada = tab[destino[0]][destino[1]]
            if capturada != "." and self.tipo(capturada) == "K":
                continue

            if not self.em_xeque(cor, novo):
                legais.append(destino)

        return legais

    def todos_movimentos(self, cor, tab):
        lista = []
        for l in range(8):
            for c in range(8):
                if tab[l][c] != "." and self.cor(tab[l][c]) == cor:
                    for destino in self.movimentos_legais(l, c, tab):
                        lista.append(((l, c), destino))
        return lista

    # --------------------------------------------------------
    # Roque
    # --------------------------------------------------------

    def pode_rocar(self, cor, lado, tab):
        l = 7 if cor == "w" else 0
        rei = "K" if cor == "w" else "k"
        torre = "R" if cor == "w" else "r"

        if tab[l][4] != rei or self.em_xeque(cor, tab):
            return False

        if lado == "k":
            if not self.roque[cor + "K"] or tab[l][7] != torre:
                return False
            if tab[l][5] != "." or tab[l][6] != ".":
                return False
            if self.casa_atacada(l, 5, "b" if cor == "w" else "w", tab):
                return False
            if self.casa_atacada(l, 6, "b" if cor == "w" else "w", tab):
                return False
            return True

        if lado == "q":
            if not self.roque[cor + "K"] or tab[l][0] != torre:
                return False
            if tab[l][1] != "." or tab[l][2] != "." or tab[l][3] != ".":
                return False
            if self.casa_atacada(l, 3, "b" if cor == "w" else "w", tab):
                return False
            if self.casa_atacada(l, 2, "b" if cor == "w" else "w", tab):
                return False
            return True

        return False

    # --------------------------------------------------------
    # Jogadas
    # --------------------------------------------------------

    def copiar_tab(self, tab):
        return [linha[:] for linha in tab]

    def aplicar_movimento_simples(self, tab, origem, destino):
        ol, oc = origem
        dl, dc = destino
        p = tab[ol][oc]
        tab[dl][dc] = p
        tab[ol][oc] = "."

        # Promoção automática para dama.
        if p == "P" and dl == 0:
            tab[dl][dc] = "Q"
        elif p == "p" and dl == 7:
            tab[dl][dc] = "q"

    def fazer_jogada(self, origem, destino):
        ol, oc = origem
        dl, dc = destino
        p = self.tabuleiro[ol][oc]

        # Atualiza direitos de roque.
        if p == "K":
            self.roque["wK"] = False
        elif p == "k":
            self.roque["bK"] = False
        elif p == "R":
            if origem == (7, 0):
                self.roque["wRq"] = False
            elif origem == (7, 7):
                self.roque["wRk"] = False
        elif p == "r":
            if origem == (0, 0):
                self.roque["bRq"] = False
            elif origem == (0, 7):
                self.roque["bRk"] = False

        # Se capturar torre na casa inicial, perde o roque.
        capturada = self.tabuleiro[dl][dc]
        if capturada == "R":
            if destino == (7, 0):
                self.roque["wRq"] = False
            elif destino == (7, 7):
                self.roque["wRk"] = False
        elif capturada == "r":
            if destino == (0, 0):
                self.roque["bRq"] = False
            elif destino == (0, 7):
                self.roque["bRk"] = False

        # Roque.
        if self.tipo(p) == "K" and abs(dc - oc) == 2:
            self.aplicar_movimento_simples(self.tabuleiro, origem, destino)

            if dc == 6:
                self.aplicar_movimento_simples(
                    self.tabuleiro, (ol, 7), (ol, 5)
                )
            else:
                self.aplicar_movimento_simples(
                    self.tabuleiro, (ol, 0), (ol, 3)
                )
        else:
            self.aplicar_movimento_simples(self.tabuleiro, origem, destino)

    def verificar_fim(self):
        movimentos = self.todos_movimentos(self.turno, self.tabuleiro)

        if not movimentos:
            self.fim = True
            if self.em_xeque(self.turno, self.tabuleiro):
                vencedor = "Brancas" if self.turno == "b" else "Pretas"
                self.status.config(text=f"Xeque-mate! {vencedor} venceu.")
                messagebox.showinfo("Fim de jogo", f"Xeque-mate!\n{vencedor} venceu!")
            else:
                self.status.config(text="Empate por afogamento.")
                messagebox.showinfo("Fim de jogo", "Empate por afogamento.")
            return True

        return False

    # --------------------------------------------------------
    # BOT
    # --------------------------------------------------------

    def valor_posicional(self, p, l, c):
        # Pequena preferência por centro, desenvolvimento e peões avançados.
        t = self.tipo(p)
        cor = self.cor(p)

        centro = 0
        if (l, c) in [(3, 3), (3, 4), (4, 3), (4, 4)]:
            centro = 18
        elif 2 <= l <= 5 and 2 <= c <= 5:
            centro = 7

        bonus = 0
        if t == "P":
            bonus = (6 - l) * 4 if cor == "w" else (l - 1) * 4
        elif t == "N" or t == "B":
            bonus = centro
        elif t == "Q":
            bonus = centro // 2

        return bonus

    def avaliar(self, tab):
        score = 0

        for l in range(8):
            for c in range(8):
                p = tab[l][c]
                if p == ".":
                    continue

                valor = VALOR[self.tipo(p)] + self.valor_posicional(p, l, c)
                if self.cor(p) == "b":
                    score += valor
                else:
                    score -= valor

        # Pequeno bônus/penalidade por xeque.
        if self.em_xeque("w", tab):
            score += 35
        if self.em_xeque("b", tab):
            score -= 35

        return score

    def jogada_bot(self):
        if self.fim:
            return

        movimentos = self.todos_movimentos("b", self.tabuleiro)

        if not movimentos:
            self.verificar_fim()
            return

        # Bot de força baixa: olha 1 lance à frente e usa
        # uma pitada de aleatoriedade. Isso evita jogar como máquina.
        candidatos = []

        for origem, destino in movimentos:
            novo = self.copiar_tab(self.tabuleiro)
            self.aplicar_movimento_simples(novo, origem, destino)

            score = self.avaliar(novo)

            capturada = self.tabuleiro[destino[0]][destino[1]]
            if capturada != ".":
                score += VALOR[self.tipo(capturada)] * 0.12

            # Penaliza um pouco movimentos de dama muito cedo.
            p = self.tabuleiro[origem[0]][origem[1]]
            if self.tipo(p) == "Q":
                score -= 10

            score += random.uniform(-45, 45)
            candidatos.append((score, origem, destino))

        candidatos.sort(reverse=True, key=lambda x: x[0])

        # Escolhe entre os melhores, simulando um jogador iniciante.
        quantidade = min(4, len(candidatos))
        escolhido = random.choice(candidatos[:quantidade])

        _, origem, destino = escolhido

        self.fazer_jogada(origem, destino)
        self.turno = "w"
        self.desenhar()

        if not self.verificar_fim():
            if self.em_xeque("w", self.tabuleiro):
                self.status.config(text="Sua vez: XEQUE!")
            else:
                self.status.config(text="Sua vez (brancas)")


if __name__ == "__main__":
    root = tk.Tk()
    app = Xadrez(root)
    root.mainloop()
