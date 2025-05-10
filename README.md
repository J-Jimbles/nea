import random
import time
import sqlite3

# Main player class for character stats
class Character:
    def __init__(self, name, char_class, hp, attack):
        self.name = name
        self.char_class = char_class
        self.max_hp = hp
        self.hp = hp
        self.attack = attack
        self.gold = 20
        self.inventory = []
        self.bosses_killed = 0

    def is_alive(self):
        return self.hp > 0

    def take_damage(self, dmg):
        self.hp -= dmg

    def heal(self, amount):
        if self.hp + amount > self.max_hp:
            self.hp = self.max_hp
        else:
            self.hp += amount

    def add_gold(self, amt):
        self.gold += amt

    def spend_gold(self, cost):
        if self.gold >= cost:
            self.gold -= cost
            return True
        return False

    def buff_attack(self, amount):
        self.attack += amount

    def show_stats(self):
        print(f"{self.name} the {self.char_class}")
        print(f"HP: {self.hp}/{self.max_hp}")
        print(f"ATK: {self.attack}")
        print(f"Gold: {self.gold}")
        print(f"Bosses Slain: {self.bosses_killed}")

    def load_inventory(self, db):
        cursor = db.execute("SELECT item_name, item_type, value, description FROM inventory WHERE player_name = ?", (self.name,))
        for row in cursor:
            item_name, item_type, value, description = row
            item = Item(item_name, item_type, value, 0, description)  # Cost is not used here
            self.inventory.append(item)

    def save_inventory(self, db):
        db.execute("DELETE FROM inventory WHERE player_name = ?", (self.name,))
        for item in self.inventory:
            db.execute("INSERT INTO inventory (player_name, item_name, item_type, value, description) VALUES (?, ?, ?, ?, ?)",
                       (self.name, item.name, item.item_type, item.value, item.desc))
        db.commit()

# Inventory management
def use_inventory(self):
    if not self.inventory:
        print("No items in inventory.")
        return

    print("\nYour items:")
    for idx, item in enumerate(self.inventory):
        print(f"{idx + 1}. {item.name} - {item.description}")

    pick = input("Use which item? (number or cancel): ").lower()
    if pick in ["cancel", "c"]:
        return

    try:
        index = int(pick) - 1
        if 0 <= index < len(self.inventory):
            item = self.inventory.pop(index)
            if item.item_type == "heal":
                item.use(self)
            else:
                print("Cannot use that item.")
    except (ValueError, IndexError):
        print("Invalid selection.")

# Enemy class
class Enemy(Character):
    def __init__(self, name, hp, attack, gold_drop, is_boss=False):
        super().__init__(name, "Enemy", hp, attack)
        self.loot = gold_drop
        self.boss = is_boss

# Item class
class Item:
    def __init__(self, name, item_type, value, cost, desc):
        self.name = name
        self.item_type = item_type
        self.value = value
        self.cost = cost
        self.desc = desc

    def use(self, char):
        if self.item_type == "heal":
            char.heal(self.value)
            print(f"Used {self.name}! Healed for {self.value} HP.")
        elif self.item_type == "weapon":
            char.buff_attack(self.value)
            char.inventory.append(self.name)
            print(f"{self.name} gives +{self.value} attack.")

# Training room for character improvement
class TrainingRoom:
    class TrainingDummy(Enemy):
        def __init__(self):
            super().__init__("Wooden Dummy", 10, 2, 0)

    def enter(self, player):
        print("\nTraining session starts.")
        dummy = self.TrainingDummy()

        while True:
            if not player.is_alive() or not dummy.is_alive():
                break

            input("Press ENTER to attack...")

            damage = random.randint(1, player.attack)
            dummy.take_damage(damage)
            print(f"Damage dealt: {damage} (Dummy HP: {dummy.hp})")

            if not dummy.is_alive():
                print("Dummy defeated.")
                print("Choose your reward: [health/attack]")
                pick = input("> ").lower()

                if pick in ["health", "h"]:
                    player.max_hp += 5
                    player.hp += 5
                    print("HP increased by 5.")
                elif pick in ["attack", "a"]:
                    player.buff_attack(1)
                    print("Attack increased by 1.")
                else:
                    print("No reward chosen.")
                return

            dummy_hit = random.randint(1, dummy.attack)
            player.take_damage(dummy_hit)
            print(f"Dummy hits back: {dummy_hit} damage (Your HP: {player.hp})")

        if not player.is_alive():
            print("You were defeated.")

# Inn class for healing
class Inn:
    def enter(self, player):
        print("\nYou enter a cozy inn.")
        print("Innkeeper: 'Need a bed? Just 1 gold.'")

        if input("[y/n]: ").lower().startswith('y'):
            if player.spend_gold(1):
                player.hp = player.max_hp
                print("HP restored.")
            else:
                print("Not enough gold.")
        else:
            print("Leaving the inn.")

# Shop system for purchasing items
class Vendor:
    def __init__(self):
        self.items = [
            Item("Small Potion", "heal", 10, 8, "Heals 10 HP"),
            Item("Iron Sword", "weapon", 3, 15, "Basic sword (+3 ATK)"),
            Item("Health Crystal", "heal", 20, 15, "Heals 20 HP")
        ]

    def interact(self, player):
        print("\nMerchant appears.")

        while True:
            choice = input("Want to buy something? [y/n/stats]: ").lower()

            if choice.startswith('n'):
                print("Goodbye.")
                break
            elif choice == "stats":
                player.show_stats()
            elif choice.startswith('y'):
                self.show_items()
                self.buy_item(player)

    def show_items(self):
        for i, item in enumerate(self.items, 1):
            print(f"{i}. {item.name} - {item.cost}g - {item.desc}")

    def buy_item(self, player):
        try:
            pick = int(input("Select item number: ")) - 1
            item = self.items[pick]
            if player.spend_gold(item.cost):
                item.use(player)
            else:
                print("Not enough gold.")
        except (IndexError, ValueError):
            print("Invalid selection.")

# Secret vendor class for rare items
class SecretVendor:
    def __init__(self):
        self.special_items = [
            Item("Shadow Blade", "weapon", 6, 30, "Spooky sword (+6 ATK)"),
            Item("Magic Elixir", "heal", 50, 40, "Major healing."),
            Item("Dragon Fang", "weapon", 8, 50, "Best weapon (+8 ATK)")
        ]

    def interact(self, player):
        print("\nA mysterious figure appears.")

        while True:
            choice = input("Buy something special? [y/n/stats]: ").lower()

            if choice.startswith('n'):
                print("The merchant vanishes.")
                break
            elif choice == "stats":
                player.show_stats()
            elif choice.startswith('y'):
                self.show_items()
                self.buy_item(player)

    def show_items(self):
        for i, item in enumerate(self.special_items, 1):
            print(f"{i}. {item.name} - {item.cost}g - {item.desc}")

    def buy_item(self, player):
        try:
            idx = int(input("Select item number: ")) - 1
            item = self.special_items[idx]
            if player.spend_gold(item.cost):
                item.use(player)
            else:
                print("Too expensive.")
        except (IndexError, ValueError):
            print("Invalid selection.")

# Random NPC dialogue
def npc_dialogue():
    print("An old mystic observes you.")
    print("'Seek wisdom, tales, or riddles?'")

    while True:
        print("\n[options: fate/story/dark/riddle/leave]")
        choice = input("> ").lower()

        if choice == "fate":
            print("'Destiny is written in blood.'")
        elif choice == "story":
            print("'The Tower once shone bright.'")
        elif choice == "dark":
            print("'Darkness comes for all.'")
        elif choice == "riddle":
            print("'Not yet ready.'")
        elif choice == "leave":
            print("The mystic nods.")
            break
        else:
            print("'Speak clearly.'")

# Main game class
class Game:
    def __init__(self):
        self.db = sqlite3.connect("game_save.db")
        self.db.execute('''CREATE TABLE IF NOT EXISTS inventory
            (player_name TEXT,
             item_name TEXT,
             item_type TEXT,
             value INTEGER,
             description TEXT)''')

        self.player = None
        self.intro_done = False

        self.places = {
            "village": {"desc": "spooky village", "npc": True, "fighting": False},
            "forest": {"desc": "dark forest", "npc": False, "fighting": True},
            "ruins": {"desc": "old ruins", "npc": False, "fighting": True},
            "mountains": {"desc": "creepy peaks", "npc": False, "fighting": True},
            "port": {"desc": "foggy docks", "npc": True, "fighting": True},
            "graveyard": {"desc": "scary cemetery", "npc": False, "fighting": True},
            "library": {"desc": "dusty books", "npc": True, "fighting": False},
            "tower": {"desc": "evil tower", "npc": False, "fighting": True},
            "dojo": {"desc": "training room", "npc": False, "fighting": False},
            "inn": {"desc": "cozy rest stop", "npc": True, "fighting": False},
        }

    def show_intro(self):
        print("\nDarkness creeps across Umbrafell...")
        time.sleep(1.5)

        print("Three ancient evils have awakened:")
        time.sleep(1)
        print("- The Wraith Queen")
        time.sleep(0.5)
        print("- The Hollow King")
        time.sleep(0.5)
        print("- The Flesh Prophet")
        time.sleep(1.5)

        print("\nOnly you can reach the dark tower...")
        time.sleep(1)
        print("(or die trying)")

    def make_character(self):
        name = input("Enter your name: ")

        print("\nSelect your class:")
        print("1) Knight - Tanky (HP++, ATK-)")
        print("2) Assassin - Sneaky (HP-, ATK+)")
        print("3) Mage - Glass cannon (HP--, ATK++)")

        pick = input("Choose (1/2/3): ")

        if pick == "1":
            self.player = Character(name, "Knight", 35, 5)
        elif pick == "2":
            self.player = Character(name, "Assassin", 28, 7)
        elif pick == "3":
            self.player = Character(name, "Mage", 22, 9)
        else:
            print("Invalid choice. Defaulting to Knight.")
            self.player = Character(name, "Knight", 35, 5)

        print(f"Welcome, {self.player.name}.")

    def spawn_enemy(self, place):
        bosses = {
            "forest": Enemy("Wraith Queen", 30, 8, 20, True),
            "ruins": Enemy("Hollow King", 35, 9, 25, True),
            "mountains": Enemy("Flesh Prophet", 40, 10, 30, True)
        }

        if place in bosses:
            return bosses[place]

        baddies = [
            Enemy("Hungry Ghoul", 18, 5, 10),
            Enemy("Dark Cultist", 22, 6, 12),
            Enemy("Ugly Beast", 26, 7, 15),
            Enemy("Bone Warrior", 30, 7, 18),
            Enemy("Gross Thing", 25, 8, 20)
        ]

        return random.choice(baddies)

    def npc_stuff(self):
        npc_types = ["healer", "merchant", "fortune", "talk"]

        npc = random.choice(npc_types)
        if npc == "healer":
            healing = random.randint(8, 15)
            self.player.heal(healing)
            print(f"Old lady heals you for {healing} HP.")
        elif npc == "merchant":
            shop = Vendor()
            shop.interact(self.player)
        elif npc == "fortune":
            print("Spooky voice: 'Kill the three... open the tower.'")
        elif npc == "talk":
            npc_dialogue()

    def explore(self, spot):
        print(f"\nYou arrive at {spot}.")
        print(f"Description: {self.places[spot]['desc']}")

        if random.random() < 0.05:
            special_shop = SecretVendor()
            special_shop.interact(self.player)
            return True

        if spot == "library":
            print("Nothing but dusty books.")
        elif spot == "inn":
            inn = Inn()
            inn.enter(self.player)
        elif spot == "dojo":
            training = TrainingRoom()
            training.enter(self.player)
            return True

        elif self.places[spot]["npc"] and random.random() < 0.7:
            self.npc_stuff()
        elif self.places[spot]["fighting"] and random.random() < 0.8:
            baddie = self.spawn_enemy(spot)
            return self.fight(baddie)
        else:
            print("It is quiet here.")

        return True

    def fight(self, enemy):
        print(f"\nA {enemy.name} appears!")

        while True:
            if not self.player.is_alive() or not enemy.is_alive():
                break

            input("Press ENTER to attack.")

            dmg = random.randint(1, self.player.attack)
            enemy.take_damage(dmg)
            print(f"Damage dealt: {dmg} (Enemy HP: {enemy.hp})")

            if not enemy.is_alive():
                print(f"You defeated the {enemy.name}.")
                self.player.add_gold(enemy.loot)
                print(f"Found {enemy.loot} gold.")
                if enemy.boss:
                    self.player.bosses_killed += 1
                return True

            enemy_dmg = random.randint(1, enemy.attack)
            self.player.take_damage(enemy_dmg)
            print(f"Enemy hits: {enemy_dmg} damage (Your HP: {self.player.hp})")

            if self.player.is_alive():
                print("Continue fighting or run away?")
                if input("[f/r]: ").lower().startswith('r'):
                    print("You ran away.")
                    return True

        print("You died.")
        return False

    def play(self):
        self.show_intro()
        self.make_character()
        self.player.load_inventory(self.db)

        while self.player.is_alive():
            self.player.show_stats()

            print("\nWhere to?")
            for place in self.places:
                print(f"- {place}")
            print("\nType 'stuff' to check inventory.")

            choice = input("\nWhere to? ").lower()

            if choice == "stuff":
                self.player.use_inventory()
                continue

            if choice not in self.places:
                print("Invalid location.")
                continue

            if choice == "tower" and self.player.bosses_killed < 3:
                print("Defeat the three bosses first.")
                continue

            if choice == "tower":
                print("Final boss encounter.")
                final_boss = Enemy("Ultimate Evil", 60, 12, 0, True)
                if self.fight(final_boss):
                    print(f"\nCongratulations, {self.player.name}!")
                break

            if not self.explore(choice):
                break

            self.player.save_inventory(self.db)

        self.db.close()
        print("\nGame over.")

# Start the game
if __name__ == "__main__":
    game = Game()
    game.play()
