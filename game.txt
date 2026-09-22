"""
Depths of Kharzul - a text adventure game.

How to play (in IDLE):
    1. Open this file in IDLE.
    2. Press F5 (Run > Run Module).
    3. Play in the Shell window that opens; type commands and press Enter.

You can also run it from a terminal with: python game.py
"""

import json
import os
import random
import sys

SAVE_FILE = os.path.join(os.path.dirname(os.path.abspath(__file__)), "kharzul_save.json")

# ---------------------------------------------------------------------------
# Data
# ---------------------------------------------------------------------------

ITEMS = {
    "dagger":          {"type": "weapon", "dmg": (2, 4),
                         "desc": "A small, rusty dagger. Better than nothing."},
    "iron_sword":      {"type": "weapon", "dmg": (4, 7),
                         "desc": "A well-balanced iron sword with a sharp edge."},
    "steel_shield":    {"type": "armor", "defense": 3,
                         "desc": "A sturdy steel shield that can turn aside a blow."},
    "torch":           {"type": "tool",
                         "desc": "A torch that still holds a flickering magical flame."},
    "rope":            {"type": "tool",
                         "desc": "A long coil of sturdy rope."},
    "brass_key":       {"type": "key",
                         "desc": "A tarnished brass key."},
    "cell_key":        {"type": "key",
                         "desc": "A small iron key that smells of mildew."},
    "vault_key":       {"type": "key",
                         "desc": "An ornate golden key, warm to the touch."},
    "healing_potion":  {"type": "potion", "heal": 10,
                         "desc": "A vial of shimmering red liquid. Restores 10 HP."},
    "cheese":          {"type": "food", "heal": 3,
                         "desc": "A wedge of surprisingly edible cheese. Restores 3 HP."},
    "stale_bread":     {"type": "food", "heal": 2,
                         "desc": "Hard as a rock, but edible. Restores 2 HP."},
    "silver_locket":   {"type": "treasure", "value": 25,
                         "desc": "A tarnished silver locket with a faded portrait inside."},
    "heart_of_kharzul": {"type": "treasure", "value": 100,
                          "desc": "A fist-sized jewel that pulses with ancient light. "
                                  "The Heart of Kharzul itself."},
}

ITEM_ALIASES = {
    "sword": "iron_sword", "shield": "steel_shield", "potion": "healing_potion",
    "key": None,  # too ambiguous, handled specially
    "locket": "silver_locket", "heart": "heart_of_kharzul", "jewel": "heart_of_kharzul",
    "bread": "stale_bread",
}

ENEMIES = {
    "giant_rat": {
        "name": "Giant Rat", "hp": 6, "dmg": (1, 3),
        "flee_chance": 0.8,
        "intro": "A mangy, oversized rat hisses and bares its yellow teeth!",
        "loot": [],
    },
    "skeleton_warrior": {
        "name": "Skeleton Warrior", "hp": 13, "dmg": (2, 5),
        "flee_chance": 0.6,
        "intro": "A skeleton in rusted mail rattles upright, raising a notched blade!",
        "loot": ["brass_key"],
    },
    "cave_serpent": {
        "name": "Cave Serpent", "hp": 15, "dmg": (3, 6),
        "flee_chance": 0.5,
        "intro": "A pale serpent uncoils from the black water, fangs glistening!",
        "loot": [],
    },
    "guardian": {
        "name": "Guardian", "hp": 32, "dmg": (4, 8),
        "flee_chance": 0.3,
        "intro": "A towering suit of animated armor grinds to life, "
                 "blocking the way out with a massive warhammer!",
        "loot": [],
    },
}

RIDDLE_QUESTION = (
    "Carved into the pedestal you read:\n"
    "  \"The more you take, the more you leave behind.\n"
    "   What am I?\""
)
RIDDLE_ANSWERS = {"footsteps", "footstep", "footprints", "footprint"}

ROOMS = {
    "entrance_hall": {
        "name": "Entrance Hall",
        "desc": "You stand in the crumbling entrance hall of Kharzul, an ancient "
                "dungeon rumoured to hold the Heart of Kharzul, a jewel of immense "
                "power. Broken statues line the walls and a cold draft flows in "
                "from the north.",
        "exits": {"north": "guard_room"},
        "items": [],
        "enemy": None,
    },
    "guard_room": {
        "name": "Guard Room",
        "desc": "Rusted weapon racks line this room. Something is slumped in the "
                "corner shadows.",
        "exits": {"south": "entrance_hall", "north": "corridor",
                  "west": "old_kitchen", "east": "armory"},
        "items": [],
        "enemy": "skeleton_warrior",
    },
    "old_kitchen": {
        "name": "Old Kitchen",
        "desc": "A ruined kitchen, thick with dust. Cookware hangs crookedly "
                "from the ceiling and something skitters in the debris.",
        "exits": {"east": "guard_room", "south": "pantry"},
        "items": ["torch", "rope"],
        "enemy": "giant_rat",
    },
    "pantry": {
        "name": "Pantry",
        "desc": "Shelves of long-spoiled food line this small pantry. A few "
                "things still look salvageable.",
        "exits": {"north": "old_kitchen"},
        "items": ["healing_potion", "cheese"],
        "enemy": None,
    },
    "armory": {
        "name": "Armory",
        "desc": "Racks of old but serviceable weapons and armor stand against "
                "the walls.",
        "exits": {"west": "guard_room"},
        "items": ["iron_sword", "steel_shield"],
        "enemy": None,
    },
    "corridor": {
        "name": "Collapsed Corridor",
        "desc": "A long corridor, half caved-in. It is pitch black here.",
        "desc_lit": "A long corridor, half caved-in. Your torchlight reveals "
                    "rubble underfoot -- and something small glinting near the wall.",
        "exits": {"south": "guard_room", "north": "library", "east": "well_room"},
        "items": [],
        "enemy": None,
    },
    "library": {
        "name": "Ruined Library",
        "desc": "Rotted bookshelves surround a stone pedestal in the centre of "
                "the room, inscribed with old text.",
        "exits": {"south": "corridor", "north": "prisoners_cell"},
        "items": [],
        "enemy": None,
    },
    "prisoners_cell": {
        "name": "Prisoner's Cell",
        "desc": "A small, filthy cell. Chained to the far wall is a gaunt "
                "figure who raises their head as you enter.",
        "exits": {"south": "library"},
        "items": [],
        "enemy": None,
    },
    "well_room": {
        "name": "Well Room",
        "desc": "An old stone well sinks into darkness at the centre of this "
                "round chamber. You can hear water far below.",
        "exits": {"west": "corridor", "down": "underground_lake"},
        "items": [],
        "enemy": None,
    },
    "underground_lake": {
        "name": "Underground Lake",
        "desc": "A vast black lake stretches out before you, lit only by "
                "faint glowing fungus on the cavern ceiling.",
        "exits": {"up": "well_room", "east": "vault_antechamber"},
        "items": [],
        "enemy": "cave_serpent",
    },
    "vault_antechamber": {
        "name": "Vault Antechamber",
        "desc": "A short passage ends in a heavy golden door, sealed tight.",
        "exits": {"west": "underground_lake", "north": "treasure_vault"},
        "items": [],
        "enemy": None,
    },
    "treasure_vault": {
        "name": "Treasure Vault",
        "desc": "Gold coins are scattered across the floor, long since picked "
                "over -- but on a stone plinth at the centre rests something "
                "far more valuable.",
        "exits": {"south": "vault_antechamber", "north": "throne_room"},
        "items": ["heart_of_kharzul"],
        "enemy": None,
    },
    "throne_room": {
        "name": "Throne Room",
        "desc": "A vast, ruined throne room. Moonlight spills through a "
                "collapsed section of ceiling far to the east, where a tunnel "
                "leads outside.",
        "exits": {"south": "treasure_vault", "east": "escape_tunnel"},
        "items": [],
        "enemy": "guardian",
    },
    "escape_tunnel": {
        "name": "Escape Tunnel",
        "desc": "Fresh air and moonlight. You made it out.",
        "exits": {},
        "items": [],
        "enemy": None,
    },
}

LOCKED_DOORS = {
    ("guard_room", "east"): {"key": "brass_key", "label": "the armory door"},
    ("library", "north"): {"key": "cell_key", "label": "the cell door"},
    ("underground_lake", "east"): {"key": "vault_key", "label": "the vault door"},
    ("throne_room", "east"): {"key": None, "flag": "guardian_defeated",
                               "label": "the sealed gate",
                               "locked_msg": "The Guardian bars the way east!"},
}

DIRECTION_ALIASES = {
    "n": "north", "s": "south", "e": "east", "w": "west",
    "u": "up", "d": "down",
    "north": "north", "south": "south", "east": "east", "west": "west",
    "up": "up", "down": "down",
}


def resolve_item_name(word):
    word = word.lower().strip()
    if word in ITEMS:
        return word
    if word in ITEM_ALIASES and ITEM_ALIASES[word]:
        return ITEM_ALIASES[word]
    for item_id in ITEMS:
        if item_id.replace("_", " ") == word:
            return item_id
    return None


# ---------------------------------------------------------------------------
# Player
# ---------------------------------------------------------------------------

class Player:
    def __init__(self):
        self.max_hp = 20
        self.hp = 20
        self.location = "entrance_hall"
        self.inventory = ["dagger"]
        self.weapon = "dagger"
        self.shield = None
        self.score = 0
        self.visited = {"entrance_hall"}
        self.previous_location = None

    def defense(self):
        if self.shield and self.shield in ITEMS:
            return ITEMS[self.shield]["defense"]
        return 0

    def weapon_damage(self):
        low, high = ITEMS[self.weapon]["dmg"]
        return random.randint(low, high)

    def heal(self, amount):
        before = self.hp
        self.hp = min(self.max_hp, self.hp + amount)
        return self.hp - before

    def to_dict(self):
        return {
            "max_hp": self.max_hp, "hp": self.hp, "location": self.location,
            "inventory": self.inventory, "weapon": self.weapon,
            "shield": self.shield, "score": self.score,
            "visited": list(self.visited),
            "previous_location": self.previous_location,
        }

    @classmethod
    def from_dict(cls, data):
        p = cls()
        p.max_hp = data["max_hp"]
        p.hp = data["hp"]
        p.location = data["location"]
        p.inventory = data["inventory"]
        p.weapon = data["weapon"]
        p.shield = data["shield"]
        p.score = data["score"]
        p.visited = set(data["visited"])
        p.previous_location = data.get("previous_location")
        return p


# ---------------------------------------------------------------------------
# Game
# ---------------------------------------------------------------------------

class Game:
    def __init__(self):
        self.player = Player()
        self.room_items = {rid: list(r["items"]) for rid, r in ROOMS.items()}
        self.enemies_alive = {rid: r["enemy"] for rid, r in ROOMS.items() if r["enemy"]}
        self.flags = {
            "riddle_solved": False,
            "prisoner_freed": False,
            "guardian_defeated": False,
            "corridor_locket_revealed": False,
        }
        self.running = True
        self.game_over = False

    # -- persistence ------------------------------------------------------

    def save(self):
        data = {
            "player": self.player.to_dict(),
            "room_items": self.room_items,
            "enemies_alive": self.enemies_alive,
            "flags": self.flags,
        }
        try:
            with open(SAVE_FILE, "w") as f:
                json.dump(data, f, indent=2)
            print("Game saved.")
        except OSError as exc:
            print("Could not save game: {}".format(exc))

    def load(self):
        if not os.path.exists(SAVE_FILE):
            print("No save file found.")
            return False
        try:
            with open(SAVE_FILE) as f:
                data = json.load(f)
            self.player = Player.from_dict(data["player"])
            self.room_items = data["room_items"]
            self.enemies_alive = data["enemies_alive"]
            self.flags = data["flags"]
            print("Game loaded.")
            return True
        except (OSError, json.JSONDecodeError, KeyError) as exc:
            print("Could not load save file: {}".format(exc))
            return False

    # -- description helpers ----------------------------------------------

    def current_room(self):
        return ROOMS[self.player.location]

    def room_description(self):
        room = self.current_room()
        rid = self.player.location
        if rid == "corridor":
            if "torch" in self.player.inventory:
                text = room["desc_lit"]
            else:
                text = room["desc"]
        else:
            text = room["desc"]
        return text

    def exits_line(self):
        room = self.current_room()
        parts = []
        for direction, target in room["exits"].items():
            key = (self.player.location, direction)
            if key in LOCKED_DOORS and not self._door_open(key):
                parts.append("{} (locked)".format(direction))
            else:
                parts.append(direction)
        return "Exits: " + ", ".join(parts) if parts else "Exits: none"

    def _door_open(self, key):
        info = LOCKED_DOORS[key]
        if info.get("flag"):
            return self.flags.get(info["flag"], False)
        return info["key"] in self.player.inventory

    def look(self):
        rid = self.player.location
        room = self.current_room()
        print()
        print("== {} ==".format(room["name"]))
        print(self.room_description())

        if rid == "corridor" and "torch" in self.player.inventory \
                and not self.flags["corridor_locket_revealed"]:
            self.room_items.setdefault("corridor", []).append("silver_locket")
            self.flags["corridor_locket_revealed"] = True
            print("By torchlight you spot something glinting among the rubble!")

        items_here = list(self.room_items.get(rid, []))
        if items_here:
            print("You see: " + ", ".join(i.replace("_", " ") for i in items_here))

        enemy_id = self.enemies_alive.get(rid)
        if enemy_id:
            print("*** {} is here! ***".format(ENEMIES[enemy_id]["name"]))

        if rid == "prisoners_cell" and not self.flags["prisoner_freed"]:
            print("The prisoner watches you warily, still in chains.")
        elif rid == "prisoners_cell" and self.flags["prisoner_freed"]:
            print("The cell is empty now save for broken chains.")

        if rid == "library" and not self.flags["riddle_solved"]:
            print("A stone pedestal sits in the centre of the room, "
                  "covered in carved text. (try: examine pedestal)")

        print(self.exits_line())

    # -- movement -----------------------------------------------------------

    def move(self, direction):
        direction = DIRECTION_ALIASES.get(direction, direction)
        room = self.current_room()
        if direction not in room["exits"]:
            print("You can't go that way.")
            return

        rid = self.player.location
        target = room["exits"][direction]
        key = (rid, direction)

        if key in LOCKED_DOORS and not self._door_open(key):
            info = LOCKED_DOORS[key]
            if info.get("flag"):
                print(info.get("locked_msg", "It's locked."))
            else:
                print("{} is locked. You'll need a key.".format(
                    info["label"].capitalize()))
            return

        enemy_id = self.enemies_alive.get(rid)
        if enemy_id:
            print("You can't leave -- {} is attacking you!".format(
                ENEMIES[enemy_id]["name"]))
            return

        if rid == "well_room" and direction == "down" and "rope" not in self.player.inventory:
            dmg = random.randint(2, 5)
            self.player.hp -= dmg
            print("With no rope, you slip and slide down the well shaft, "
                  "scraping yourself badly! (-{} HP)".format(dmg))
            if self._check_death():
                return

        self._enter_room(target)

    def _enter_room(self, target, trigger_ambush=True):
        """Move the player into `target`, applying entry hazards, then describe
        it and (optionally) let a waiting enemy ambush the player."""
        self.player.previous_location = self.player.location
        self.player.location = target
        if target not in self.player.visited:
            self.player.visited.add(target)
            self.player.score += 5

        if target == "corridor" and "torch" not in self.player.inventory:
            dmg = random.randint(1, 3)
            self.player.hp -= dmg
            print("It's pitch black. You stumble over debris in the dark! (-{} HP)".format(dmg))
            if self._check_death():
                return

        if target == "escape_tunnel":
            self._win()
            return

        self.look()
        if trigger_ambush:
            self.maybe_start_combat()

    # -- inventory / items ----------------------------------------------

    def take(self, word):
        if not word:
            print("Take what?")
            return
        item_id = resolve_item_name(word)
        rid = self.player.location
        here = self.room_items.get(rid, [])
        if item_id and item_id in here:
            here.remove(item_id)
            self.player.inventory.append(item_id)
            print("You take the {}.".format(item_id.replace("_", " ")))
            if ITEMS[item_id]["type"] == "treasure":
                self.player.score += ITEMS[item_id]["value"]
        else:
            # allow matching by loose text even if not in ITEM_ALIASES
            match = None
            for candidate in here:
                if word in candidate.replace("_", " "):
                    match = candidate
                    break
            if match:
                here.remove(match)
                self.player.inventory.append(match)
                print("You take the {}.".format(match.replace("_", " ")))
                if ITEMS[match]["type"] == "treasure":
                    self.player.score += ITEMS[match]["value"]
            else:
                print("There's no {} here.".format(word))

    def drop(self, word):
        if not word:
            print("Drop what?")
            return
        item_id = resolve_item_name(word)
        if item_id not in self.player.inventory:
            for candidate in self.player.inventory:
                if word in candidate.replace("_", " "):
                    item_id = candidate
                    break
        if item_id in self.player.inventory:
            self.player.inventory.remove(item_id)
            self.room_items.setdefault(self.player.location, []).append(item_id)
            if self.player.weapon == item_id:
                self.player.weapon = "dagger"
            if self.player.shield == item_id:
                self.player.shield = None
            print("You drop the {}.".format(item_id.replace("_", " ")))
        else:
            print("You don't have that.")

    def show_inventory(self):
        print()
        print("HP: {}/{}".format(self.player.hp, self.player.max_hp))
        print("Weapon: {}".format(self.player.weapon.replace("_", " ")))
        print("Shield: {}".format(self.player.shield.replace("_", " ")
                                   if self.player.shield else "none"))
        if not self.player.inventory:
            print("Inventory: (empty)")
        else:
            print("Inventory:")
            for item_id in self.player.inventory:
                print("  - {}: {}".format(item_id.replace("_", " "), ITEMS[item_id]["desc"]))
        print("Score: {}".format(self.player.score))

    def equip(self, word):
        item_id = resolve_item_name(word) or word
        if item_id not in self.player.inventory:
            print("You don't have that.")
            return
        kind = ITEMS.get(item_id, {}).get("type")
        if kind == "weapon":
            self.player.weapon = item_id
            print("You equip the {}.".format(item_id.replace("_", " ")))
        elif kind == "armor":
            self.player.shield = item_id
            print("You equip the {}.".format(item_id.replace("_", " ")))
        else:
            print("You can't equip that.")

    def use_item(self, word, in_combat_enemy=None):
        item_id = resolve_item_name(word)
        if item_id not in self.player.inventory:
            for candidate in self.player.inventory:
                if word in candidate.replace("_", " "):
                    item_id = candidate
                    break
        if item_id not in self.player.inventory:
            print("You don't have that.")
            return False

        info = ITEMS[item_id]
        if info["type"] in ("potion", "food"):
            healed = self.player.heal(info["heal"])
            self.player.inventory.remove(item_id)
            print("You consume the {} and recover {} HP. ({}/{})".format(
                item_id.replace("_", " "), healed, self.player.hp, self.player.max_hp))
            return True
        elif info["type"] == "weapon":
            self.equip(item_id)
            return True
        elif info["type"] == "armor":
            self.equip(item_id)
            return True
        else:
            print("Nothing happens.")
            return False

    # -- examine / talk / riddle -------------------------------------------

    def examine(self, word):
        rid = self.player.location
        if not word:
            print(self.room_description())
            return
        if rid == "library" and word in ("pedestal", "podium", "inscription", "text"):
            print(RIDDLE_QUESTION)
            if self.flags["riddle_solved"]:
                print("(You've already solved this.)")
            return
        if rid == "prisoners_cell" and word in ("prisoner", "man", "woman", "figure"):
            print("A gaunt figure in tattered robes, chained to the wall. "
                  "They look like they've been here a long while.")
            return
        item_id = resolve_item_name(word)
        if item_id and item_id in self.player.inventory:
            print(ITEMS[item_id]["desc"])
            return
        here = self.room_items.get(rid, [])
        for candidate in here:
            if word in candidate.replace("_", " "):
                print(ITEMS[candidate]["desc"])
                return
        print("You see nothing special about that.")

    def answer(self, text):
        rid = self.player.location
        if rid != "library":
            print("There's nothing to answer here.")
            return
        if self.flags["riddle_solved"]:
            print("The pedestal has nothing more to offer.")
            return
        if text.lower().strip() in RIDDLE_ANSWERS:
            print("The pedestal glows faintly. A hidden drawer slides open, "
                  "revealing a small iron key!")
            self.flags["riddle_solved"] = True
            self.player.inventory.append("cell_key")
            self.player.score += 15
        else:
            print("The pedestal remains silent. That isn't it.")

    def talk(self, word):
        rid = self.player.location
        if rid != "prisoners_cell":
            print("There's no one here to talk to.")
            return
        if not self.flags["prisoner_freed"]:
            print('The prisoner rasps: "You found the key... thank the stars. '
                  'Listen -- deep below, past the lake, lies a vault. Take this, '
                  'it should open the door. And beware what you wake when you '
                  'take what\'s inside."')
            print("The prisoner presses a golden key into your hand before "
                  "slipping away into the dark corridors.")
            self.flags["prisoner_freed"] = True
            self.player.inventory.append("vault_key")
            self.player.score += 10
        else:
            print("The cell is empty. The prisoner is long gone.")

    # -- combat -------------------------------------------------------------

    def maybe_start_combat(self):
        rid = self.player.location
        enemy_id = self.enemies_alive.get(rid)
        if enemy_id:
            self.combat(enemy_id)

    def combat(self, enemy_id):
        template = ENEMIES[enemy_id]
        enemy_hp = template["hp"]
        print()
        print(template["intro"])

        while self.running and enemy_hp > 0 and self.player.hp > 0:
            print()
            print("Your HP: {}/{}   {} HP: {}".format(
                self.player.hp, self.player.max_hp, template["name"], enemy_hp))
            action = input("(attack / defend / use <item> / flee) > ").strip().lower()

            defending = False
            if action in ("attack", "a", "fight"):
                dmg = self.player.weapon_damage()
                enemy_hp -= dmg
                print("You strike the {} for {} damage.".format(template["name"], dmg))
            elif action in ("defend", "d", "block"):
                defending = True
                print("You brace yourself for the next attack.")
            elif action.startswith("use "):
                self.use_item(action[4:].strip())
            elif action in ("flee", "run", "f"):
                if random.random() < template["flee_chance"]:
                    retreat_to = self.player.previous_location
                    if retreat_to not in ROOMS:
                        retreat_to = "entrance_hall"
                    print("You break away and flee back toward the {}!".format(
                        ROOMS[retreat_to]["name"]))
                    self._enter_room(retreat_to, trigger_ambush=False)
                    return
                else:
                    print("You can't get away!")
            else:
                print("You're not sure how to do that. "
                      "(attack / defend / use <item> / flee)")
                continue

            if enemy_hp <= 0:
                break

            raw_dmg = random.randint(*template["dmg"])
            reduction = self.player.defense()
            if defending:
                raw_dmg = raw_dmg // 2
            dmg_taken = max(0, raw_dmg - reduction)
            self.player.hp -= dmg_taken
            print("The {} hits you for {} damage.".format(template["name"], dmg_taken))

            if self._check_death():
                return

        if enemy_hp <= 0:
            print()
            print("You have defeated the {}!".format(template["name"]))
            self.player.score += 20
            del self.enemies_alive[self.player.location]
            for loot in template["loot"]:
                if loot not in self.player.inventory:
                    self.player.inventory.append(loot)
                    print("The {} dropped a {}.".format(
                        template["name"], loot.replace("_", " ")))
            if enemy_id == "guardian":
                self.flags["guardian_defeated"] = True
                print("The suit of armor crashes to the ground, unmoving. "
                      "The way east is finally clear.")

    def _check_death(self):
        if self.player.hp <= 0:
            self.player.hp = 0
            print()
            print("Everything fades to black...")
            print("You have died in the Depths of Kharzul.")
            print("Final score: {}".format(self.player.score))
            self.game_over = True
            self.running = False
            return True
        return False

    def _win(self):
        print()
        print("== ESCAPE TUNNEL ==")
        print(self.room_description())
        has_heart = "heart_of_kharzul" in self.player.inventory
        print()
        if has_heart:
            print("You stumble out into the moonlight, the Heart of Kharzul "
                  "blazing in your pack. Its light seems to promise a very "
                  "different life from here on.")
            print("*** TRUE ENDING: You escaped the Depths of Kharzul with its "
                  "greatest treasure! ***")
        else:
            print("You stumble out into the moonlight, bruised but alive. "
                  "The Heart of Kharzul remains somewhere in the dark behind you.")
            print("*** YOU ESCAPED, but the Heart of Kharzul remains lost. ***")
        print()
        print("Rooms explored: {}/{}".format(len(self.player.visited), len(ROOMS)))
        print("Final score: {}".format(self.player.score))
        self.game_over = True
        self.running = False

    # -- command handling -----------------------------------------------

    def help_text(self):
        print()
        print("Commands:")
        print("  go <direction>   (or just: north/n, south/s, east/e, west/w, up/u, down/d)")
        print("  look / l         - describe the room again")
        print("  inventory / i    - show your items, HP and equipment")
        print("  take <item>      - pick up an item")
        print("  drop <item>      - drop an item")
        print("  use <item>       - drink a potion, eat food, equip a weapon/shield")
        print("  equip <item>     - wield a weapon or raise a shield")
        print("  examine <thing>  - look closely at an item or feature")
        print("  talk <person>    - speak to someone in the room")
        print("  answer <text>    - answer a riddle")
        print("  attack           - attack an enemy in the room")
        print("  save / load      - save or load your progress")
        print("  help             - show this list")
        print("  quit             - exit the game")

    def handle_command(self, raw):
        raw = raw.strip()
        if not raw:
            return
        parts = raw.split(None, 1)
        verb = parts[0].lower()
        rest = parts[1].strip() if len(parts) > 1 else ""

        if verb in DIRECTION_ALIASES:
            self.move(DIRECTION_ALIASES[verb])
        elif verb in ("go", "move", "walk"):
            if not rest:
                print("Go where?")
            else:
                self.move(DIRECTION_ALIASES.get(rest.lower(), rest.lower()))
        elif verb in ("look", "l"):
            self.look()
        elif verb in ("inventory", "inv", "i"):
            self.show_inventory()
        elif verb in ("take", "get", "grab", "pickup"):
            self.take(rest.lower())
        elif verb == "drop":
            self.drop(rest.lower())
        elif verb == "use":
            self.use_item(rest.lower())
        elif verb == "equip":
            self.equip(rest.lower())
        elif verb in ("examine", "x", "inspect"):
            self.examine(rest.lower())
        elif verb in ("talk", "speak"):
            self.talk(rest.lower())
        elif verb in ("answer", "say"):
            self.answer(rest)
        elif verb in ("attack", "fight"):
            enemy_id = self.enemies_alive.get(self.player.location)
            if enemy_id:
                self.combat(enemy_id)
            else:
                print("There's nothing here to attack.")
        elif verb == "save":
            self.save()
        elif verb == "load":
            self.load()
        elif verb in ("help", "h", "?"):
            self.help_text()
        elif verb in ("quit", "exit", "q"):
            self._quit()
        else:
            print("I don't understand '{}'. Type 'help' for a list of commands.".format(raw))

    def _quit(self):
        answer = input("Save before quitting? (y/n) > ").strip().lower()
        if answer.startswith("y"):
            self.save()
        self.running = False

    # -- main loop --------------------------------------------------------

    def run(self):
        print()
        print("=" * 60)
        print("               DEPTHS OF KHARZUL")
        print("=" * 60)
        print("Type 'help' at any time to see the list of commands.")
        self.look()
        self.maybe_start_combat()

        while self.running:
            if self.game_over:
                break
            try:
                raw = input("\n> ")
            except (EOFError, KeyboardInterrupt):
                print("\nGoodbye!")
                break
            self.handle_command(raw)
            if self.game_over:
                break


# ---------------------------------------------------------------------------
# Entry point
# ---------------------------------------------------------------------------

def main_menu():
    print("=" * 60)
    print("               DEPTHS OF KHARZUL")
    print("        a text adventure in the old dungeon style")
    print("=" * 60)
    while True:
        print()
        print("1) New Game")
        if os.path.exists(SAVE_FILE):
            print("2) Load Game")
        print("3) Quit")
        choice = input("> ").strip().lower()

        if choice in ("1", "new", "new game"):
            return Game()
        elif choice in ("2", "load", "load game") and os.path.exists(SAVE_FILE):
            game = Game()
            if game.load():
                return game
        elif choice in ("3", "quit", "exit", "q"):
            print("Goodbye!")
            sys.exit(0)
        else:
            print("Please choose a valid option.")


def main():
    game = main_menu()
    game.run()

    while game.game_over and game.player.hp <= 0:
        again = input("\nPlay again? (y/n) > ").strip().lower()
        if again.startswith("y"):
            game = Game()
            game.run()
        else:
            break

    print("\nThanks for playing Depths of Kharzul!")


if __name__ == "__main__":
    main()
