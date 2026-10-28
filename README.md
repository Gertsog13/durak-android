#!/usr/bin/env python3
"""Подкидной дурак в консоли: вы против компьютера.

Запуск:  python durak.py
"""
import random
import sys

SUITS = "♠♥♦♣"
RANK_NAMES = {6: "6", 7: "7", 8: "8", 9: "9", 10: "10", 11: "В", 12: "Д", 13: "К", 14: "Т"}
HAND_SIZE = 6


class Card:
    def __init__(self, rank, suit):
        self.rank = rank
        self.suit = suit

    def __str__(self):
        return f"{RANK_NAMES[self.rank]}{self.suit}"

    __repr__ = __str__

    def beats(self, other, trump):
        """Может ли эта карта побить карту other."""
        if self.suit == other.suit:
            return self.rank > other.rank
        return self.suit == trump and other.suit != trump


class Player:
    def __init__(self, name, human):
        self.name = name
        self.human = human
        self.hand = []

    def sort_hand(self, trump):
        self.hand.sort(key=lambda c: (c.suit == trump, c.rank, c.suit))


class Game:
    def __init__(self, auto=False):
        self.auto = auto
        self.you = Player("Компьютер 1" if auto else "Вы", not auto)
        self.cpu = Player("Компьютер 2" if auto else "Компьютер", False)
        self.deck = [Card(r, s) for r in RANK_NAMES for s in SUITS]
        random.shuffle(self.deck)
        # deck[0] — козырная карта, лежит внизу колоды; берём с конца
        self.trump_card = self.deck[0]
        self.trump = self.trump_card.suit

    # ---------- вспомогательное ----------
    def say(self, text=""):
        print(text)

    def refill(self, player):
        while len(player.hand) < HAND_SIZE and self.deck:
            player.hand.append(self.deck.pop())
        player.sort_hand(self.trump)

    def other(self, player):
        return self.cpu if player is self.you else self.you

    def first_attacker(self):
        best, who = None, None
        for p in (self.you, self.cpu):
            for c in p.hand:
                if c.suit == self.trump and (best is None or c.rank < best):
                    best, who = c.rank, p
        return who or random.choice([self.you, self.cpu])

    def show_state(self, table):
        print("\n" + "=" * 50)
        trump_info = f"Козырь: {self.trump}"
        if self.deck:
            trump_info += f" (внизу колоды {self.trump_card})"
        print(f"{trump_info}   Карт в колоде: {len(self.deck)}")
        opp = self.cpu if not self.auto else self.cpu
        print(f"У соперника карт: {len(opp.hand)}")
        print("-" * 50)
        if table:
            print("Стол:")
            for a, d in table:
                print(f"   {a}  ←  {d if d else '?'}")
        else:
            print("Стол пуст")
        print("-" * 50)

    def show_hand(self, player):
        print("Ваши карты: " + "  ".join(f"[{i}] {c}" for i, c in enumerate(player.hand, 1)))

    # ---------- ввод человека ----------
    def ask(self, prompt, valid):
        while True:
            raw = input(prompt).strip()
            if raw.isdigit() and int(raw) in valid:
                return int(raw)
            print("Не понял, попробуйте ещё раз.")

    def human_attack(self, table, first):
        ranks = {c.rank for pair in table for c in pair if c}
        options = [i for i, c in enumerate(self.you.hand, 1) if first or c.rank in ranks]
        if not options:
            return None
        self.show_hand(self.you)
        if first:
            self.say("Вы ходите. Выберите карту.")
            n = self.ask("Номер карты: ", set(options))
        else:
            self.say("Можно подкинуть карту (того же достоинства, что на столе).")
            n = self.ask("Номер карты (0 — бито/хватит): ", set(options) | {0})
        if n == 0:
            return None
        return self.you.hand.pop(n - 1)

    def human_defend(self, attack):
        options = [i for i, c in enumerate(self.you.hand, 1) if c.beats(attack, self.trump)]
        self.show_hand(self.you)
        self.say(f"Вам ходят: {attack}")
        if not options:
            input("Побить нечем — придётся брать. Нажмите Enter... ")
            return None
        n = self.ask("Чем бьёте? Номер карты (0 — взять): ", set(options) | {0})
        if n == 0:
            return None
        return self.you.hand.pop(n - 1)

    # ---------- ИИ ----------
    def ai_attack(self, player, table, first):
        ranks = {c.rank for pair in table for c in pair if c}
        cands = [c for c in player.hand if first or c.rank in ranks]
        if not cands:
            return None
        cands.sort(key=lambda c: (c.suit == self.trump, c.rank))
        card = cands[0]
        # козыри без нужды не подкидываем, пока есть колода
        if not first and card.suit == self.trump and self.deck:
            return None
        player.hand.remove(card)
        return card

    def ai_defend(self, player, attack):
        beaters = [c for c in player.hand if c.beats(attack, self.trump)]
        if not beaters:
            return None
        beaters.sort(key=lambda c: (c.suit == self.trump, c.rank))
        card = beaters[0]
        # не тратим козырь на мелочь, пока в колоде много карт
        if card.suit == self.trump and attack.suit != self.trump \
                and attack.rank <= 9 and len(self.deck) >= 4:
            return None
        player.hand.remove(card)
        return card

    # ---------- ход игрока в роли атакующего/защитника ----------
    def attack(self, player, table, first):
        if player.human:
            return self.human_attack(table, first)
        return self.ai_attack(player, table, first)

    def defend(self, player, card):
        if player.human:
            return self.human_defend(card)
        return self.ai_defend(player, card)

    # ---------- один раунд ----------
    def play_round(self, att, dfn):
        table = []  # пары [атака, защита или None]
        limit = min(HAND_SIZE, len(dfn.hand))
        taking = False

        card = self.attack(att, table, True)
        table.append([card, None])
        self.log(att, f"{att.name} ходит: {card}")

        while True:
            if not taking:
                self.show_for_human(table)
                response = self.defend(dfn, table[-1][0])
                if response is None:
                    taking = True
                    self.log(dfn, f"{dfn.name} берёт карты.")
                else:
                    table[-1][1] = response
                    self.log(dfn, f"{dfn.name} бьёт: {response}")
                    if not dfn.hand:
                        break

            if len(table) >= limit or not att.hand:
                break
            self.show_for_human(table)
            extra = self.attack(att, table, False)
            if extra is None:
                break
            table.append([extra, None])
            self.log(att, f"{att.name} подкидывает: {extra}")

        cards = [c for pair in table for c in pair if c]
        if taking:
            dfn.hand.extend(cards)
            dfn.sort_hand(self.trump)
            self.say(f"\n{dfn.name} забирает {len(cards)} карт(ы).")
        else:
            self.say("\nБито!")
        return taking

    def log(self, player, text):
        if not player.human or self.auto:
            print(text)

    def show_for_human(self, table):
        if self.auto:
            return
        self.show_state(table)

    # ---------- вся игра ----------
    def play(self):
        for _ in range(HAND_SIZE):
            self.you.hand.append(self.deck.pop())
            self.cpu.hand.append(self.deck.pop())
        self.you.sort_hand(self.trump)
        self.cpu.sort_hand(self.trump)

        print("Подкидной дурак. Козырная карта:", self.trump_card)
        att = self.first_attacker()
        dfn = self.other(att)
        print(f"Первым ходит: {att.name}")

        while True:
            taken = self.play_round(att, dfn)
            self.refill(att)
            self.refill(dfn)

            if not self.deck:
                empty = [p for p in (self.you, self.cpu) if not p.hand]
                if len(empty) == 2:
                    print("\nНичья!")
                    return None
                if empty:
                    winner = empty[0]
                    loser = self.other(winner)
                    print(f"\n{winner.name} — свободен, а {loser.name} — дурак!")
                    return winner

            if not taken:
                att, dfn = dfn, att
            if self.auto:
                print("\n" + "-" * 30)
            else:
                input("\nEnter — следующий раунд... ")


def main():
    auto = "--auto" in sys.argv
    while True:
        winner = Game(auto=auto).play()
        if auto:
            return
        if input("\nСыграем ещё? (д/н): ").strip().lower() not in ("д", "y", "да", "yes"):
            print("Спасибо за игру!")
            return


if __name__ == "__main__":
    try:
        main()
    except (KeyboardInterrupt, EOFError):
        print("\nДо встречи!")
