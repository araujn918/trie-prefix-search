# trie data structure: how prefix search really works, why lookup is O(L) instead of O(n), and the interview problems built on top of it

Most people land on a trie for one of three reasons. They're building autocomplete and a hash map just failed them. An interview question said "implement a prefix tree" and they nodded along. Or they read that a router does "longest prefix match" on IP addresses and wanted to know what that actually means.

All three come back to one design decision: don't store the key in the node. Let the path spell it.

That single idea is why a trie can answer "does any word start with `car`?" in the time it takes to read three characters, no matter whether the dictionary holds a thousand words or ten million.

## What a trie actually is

A trie is a tree where each node represents one character, and the path from the root down to a node spells out a key. The root is the empty string. Every child of a node shares the prefix that leads to that node.

The important consequence: nodes don't hold their keys. A node's *position* defines its key. That's the opposite of a binary search tree, where each node carries a full key and you compare strings along the way. In a trie, the comparison work is distributed across the structure itself, so you never compare whole strings at all.

Two other details that trip people up:

- If one key is a prefix of another (`act` and `actor`), you need an end marker. Without one, you can't distinguish "this node ends a valid word" from "this node is just a waypoint." A boolean flag on the node handles it.
- A "word" is a root-to-node path, and it may terminate on an internal node, not necessarily a leaf.

The name comes from *retrieval*. Edward Fredkin coined it in 1960 and pronounced it "tree," which was a genuinely poor decision for spoken conversation. Plenty of people say "try" instead, specifically to avoid the confusion. Both are in use; neither will get you corrected. The underlying structure is older than the name — Axel Thue described the idea abstractly back in 1912, and René de la Briandais published a computer-oriented version in 1959.

## The two operations that matter

Every trie question reduces to insert and lookup. Here's the whole thing in Python, with a dict of children:

python
class TrieNode:
    __slots__ = ("children", "is_word")
    def __init__(self):
        self.children = {}
        self.is_word = False

class Trie:
    def __init__(self):
        self.root = TrieNode()

    def insert(self, word):
        node = self.root
        for ch in word:
            node = node.children.setdefault(ch, TrieNode())
        node.is_word = True

    def search(self, word):
        node = self._walk(word)
        return node is not None and node.is_word

    def starts_with(self, prefix):
        return self._walk(prefix) is not None

    def _walk(self, s):
        node = self.root
        for ch in s:
            if ch not in node.children:
                return None
            node = node.children[ch]
        return node


Insertion walks down the characters, creating nodes as needed, and sets the end flag on the final one. Lookup walks the same way and then checks the flag. That's it.

If you want the classic textbook version instead, swap the dict for a fixed array: `self.children = [None] * 26` for lowercase English. Faster indexing, much more memory.

The interesting part is what happens on a failed lookup. Suppose your trie holds `she`, `shells`, `sea`, `sells`. You ask for `shore`. You walk `s`, then `h`, then look for `o` and find nothing. You stop after two characters, having eliminated every key that begins with `sh`. You never touched the other branches.

That early termination is the entire point. Searching for a prefix that doesn't exist costs you the length of the prefix, not the size of the dictionary.

### Why the worst case doesn't depend on n

For a successful search, you examine exactly L characters, where L is the key length. Not "on average" — always, exactly L, because you follow one link per character and there's no branching decision to make.

For a failed search, with random keys the tree tends to be about `log_R N` deep, where R is the alphabet size and N is the number of keys. Each level down cuts the remaining candidates by a factor of R. So misses are typically sublinear in the number of stored keys.

Compare that to a balanced BST, which pays `lg N` string comparisons per operation, and each comparison itself costs up to L character checks. The trie trades that away entirely.

## Where tries beat hash maps, and where they don't

A hash map is the default answer for "look up this exact string." It's simpler, it's faster in practice for plain membership tests, and it uses less memory. Most of the time, reach for it.

A hash map has no concept of order or adjacency. To find all keys starting with `pre`, you either maintain a separate sorted structure or scan every key. That's the gap a trie fills.

| Capability | Hash map | Balanced BST | R-way trie |
| --- | --- | --- | --- |
| Exact-match lookup | O(L) expected | O(L·lg N) | O(L) |
| Prefix query | Not supported without a scan | Workable, costs lg N per node | O(L), then a subtree walk |
| Iterate keys in sorted order | Sort the keys first | Free from in-order traversal | Free from lexicographic DFS |
| Wildcard pattern search | Scan | Poor fit | Good fit (branch over the wildcard) |
| Longest-prefix match | Awkward | Awkward | Natural |
| Memory per node | — | — | R links, most of them null |

The sorted-iteration row is underrated. An in-order traversal of a balanced BST gives sorted order too, so on that axis they tie. The trie's advantage is that the sorted order falls out of the structure for free while still supporting the prefix operations.

The wildcard row matters more in interviews than in production. Patterns like `b..k` or `b*ch` are painful for hash-based structures and nearly free for a trie, because a wildcard is just a node where you branch over all children instead of one.

The last row is where the trouble starts.

## The memory problem, and the three standard fixes

A plain R-way trie gives each node R links, one per possible character. For lowercase English that's 26 slots per node. For extended ASCII it's 256 pointers per node, most of which are null and doing nothing useful.

The scale of the waste is easy to see in textbook benchmark data. On the `moby.txt` word list with R = 256, an R-way trie uses about 1.12 references per character stored, which is respectable. On the `actors.txt` dataset in the same benchmark, the R-way trie *ran out of memory* while a linear-probing hash table needed only about 0.76 references per character. Same lecture notes, same machine, one structure finishes and the other doesn't.

Multiply that problem by Unicode and it stops being an optimization conversation. Roughly 65,536 possible values in the basic range, over 230,000 characters in total. A 65,536-way array per node is not a data structure, it's a mistake.

Three fixes show up over and over:

**Ternary search trie.** Each node stores a single character and three pointers: left for smaller characters, right for larger ones, and middle for the subtrie continuing with that character. It behaves like a BST on characters within each level, which cuts memory dramatically at the cost of a few extra comparisons. The size no longer grows with the alphabet size.

**Compressed trie (radix tree).** Any chain of nodes that has exactly one child gets collapsed into a single edge labeled with a substring instead of a character. When every internal node has at least two children, the number of internal nodes is bounded by the number of stored words. You stop paying for long shared prefixes character by character.

**Bitwise trie.** Skip characters entirely and key on the individual bits of fixed-length data, such as an integer or a memory address. This is the structure behind the "maximum XOR of two numbers" family of problems, where you greedily walk toward the opposite bit at each level.

There's also a lower-level trick used in production implementations: nodes built from a compact bitmap plus a pointer array, where the offset to a child pointer is computed from the number of set bits below the target position — a `popcount` operation that modern CPUs execute as a single instruction. A 64-bit bitmap matches the register width, so each node covers a 64-symbol alphabet with no wasted space for characters the node doesn't use.

## What actually runs on tries

Autocomplete and typeahead are the obvious case. Walk the prefix, then DFS the subtree to collect completions. The alternative — scanning every stored term — is fine at a few hundred entries and hopeless at a few million.

Spell checkers and word games use the same early-exit property. In Boggle-style solvers, if a letter sequence isn't a prefix of any word, you prune that entire branch of the board search immediately instead of testing the whole dictionary.

Network routers are the case people don't expect. A routing table maps IP addresses to outgoing links, and the trie is keyed on address bits. When a packet arrives, the router finds the *longest* matching prefix and forwards along that entry. Longest-prefix match is awkward in a hash table and natural in a trie, which is why routing tables have looked like tries for decades.

Search engines and text-processing pipelines use suffix tries, which are just tries built over every suffix of a text. They answer "does pattern P occur in T," "how many times does it occur," and "is P a suffix of T" — questions that matter for pattern matching over stable, slowly changing text such as source corpora and genome sequences. Suffix trees add suffix links on top so you can move between nested prefixes without re-walking from the root.

An implementation note if you go down that road: for large texts you don't store the strings in the nodes. You store a position and a length pointing into the original text and slice when needed. Storing copies of every suffix is how you turn a memory-savvy structure into a memory disaster.

## The interview version of the problem

Tries come up in coding interviews in a narrow set of shapes, and recognizing the shape matters more than memorizing code:

1. **Implement a trie.** Insert, exact search, prefix search. This is the warm-up, and it's where end markers get tested.
2. **Wildcard search.** `.` matches any single character, so the search branches across children at that position. Recursion or a queue, both fine.
3. **Word Search II.** A grid of letters plus a word list. Build a trie of the words, DFS the grid, and prune the moment you leave the trie. Brute force per-word search times out; the trie version passes.
4. **Replace words / prefix replacement.** Find the shortest root that is a prefix of a word. Walk the trie and stop at the first end marker.
5. **Search suggestions and autocomplete.** Walk the prefix node, then collect the subtree, often ranked by frequency counters stored on nodes.
6. **Maximum XOR pair.** A bitwise trie over 32-bit values, one bit per level, greedily choosing the opposite bit.
7. **Palindrome pairs.** Build a trie of reversed words and look for the complementary split points.
8. **Longest common prefix across a set of strings.** Walk the trie until a node has more than one child or an end marker.

If you're preparing and want the structure to stick, the efficient path is to write a trie from scratch twice — once with a dict of children and once with an array — before touching the problem list. The mechanics are only about twenty lines, and once they're automatic, the problems above become variations on "walk, then branch or stop."

That's also roughly the pitch behind AlgoMonster, the platform this article's links point to. It was built by engineers from Google and organized around named patterns rather than a wall of unordered practice problems, with interactive code editors in the browser, illustrated explanations, and a Speedrun mode for timed review. Third-party reviews of the platform consistently describe 48 patterns, with advanced data structures among them, plus a data structure refresher for people whose fundamentals are shaky.

The honest framing for anyone weighing it up: this is a text-and-diagram platform. There are no video walkthroughs, and the problem bank is far smaller than what LeetCode offers — you're paying for the pattern framework and the explanations, not for volume. If you learn best by watching someone type, it's the wrong product for you.

## What AlgoMonster costs right now

The platform runs three tiers. Full access sits behind the paid plans; the free tier is a genuine preview rather than a usable curriculum.

One warning before the table: AlgoMonster prices swing hard and often. Within a short window, the official subscription page has displayed different numbers across different visits — one version showed a yearly price of $69.99 ($5.84/month billed annually) with a lifetime option at $89, another showed $99/year, another $119/year, and lifetime has appeared as high as $459. That's normal for a platform that seems to run near-permanent promotions, but it means any price you read on a blog is a snapshot. Confirm on the page before you decide anything.

| Plan | Billing | What you get | Price recently displayed | Get started |
| --- | --- | --- | --- | --- |
| Free | No charge | Limited access to a subset of patterns, lessons and problems | $0 | [Create a free AlgoMonster account and look around](https://algo.monster?fpr=li69) |
| Pro – Yearly | Billed once per year, 12 months of access, cancel or upgrade anytime | All patterns, lessons and problems unlocked; unlimited AI Assistant; unlimited Speedrun; real questions from top tech companies, refreshed monthly; 500+ lessons across 6 courses, 1,000+ illustrations, 3,000+ editorials; member discount on live bootcamps | $69.99–$119/year depending on the active sale | [See the current Pro Yearly price](https://algo.monster?fpr=li69) |
| Pro – Lifetime | One-time payment | Everything in the annual plan, plus lifetime updates and permanent access | $89–$459 one-time, depending on the promotion | [Check the Pro Lifetime price](https://algo.monster?fpr=li69) |

A couple of notes that don't fit neatly in a cell.

There's no monthly option and no team or enterprise tier. It's annual or lifetime, individual only. If you need prep for a system design loop at senior level, the platform's coverage there is introductory, and several reviews point out it isn't enough for L5+ interviews on its own — budget for a separate resource.

On the other hand, the lifetime plan is genuinely cheap when it's on sale, and the break-even math is simple. If it comes in near the low end of that range, the one-time price is under what a single year of competing subscriptions costs. If it's near $459, you're betting you'll prep for interviews more than once, which is a reasonable bet for engineers who revisit the market every few years and a bad one if you expect to land a job and never open it again.

For the trie topic specifically, the free tier is enough to see whether the explanation style works for you before spending anything. If the diagrams land, the paid tiers are what unlock the full pattern set and the problem list. If you'd rather read plain textbook treatments, the Princeton and CMU lecture notes behind the complexity numbers in this article are free and excellent.

## Mistakes worth avoiding

**Forgetting the end marker.** It works fine on `car` and breaks the moment `cart` is also in the set. Every "why does my trie return true for a prefix" bug traces back here.

**Assuming constant time.** Lookup is O(L), not O(1). For dictionary words that's effectively constant. For genome-length sequences, L is the dominant cost and a different structure is probably the right call.

**Using a 26-slot array for arbitrary input.** Fine for a lowercase-only interview problem, wrong for anything user-facing with Unicode, accents, or emoji.

**Deleting keys by nulling nodes.** If `car` and `cart` share the `c-a-r` path, deleting `car` must not remove the nodes that `cart` still needs. Either track a count of words passing through each node, or unset the end flag and clean up only the nodes that become childless on the way back up.

**Reaching for a trie when you need exact matches only.** A hash map is simpler, faster, and lighter. The trie earns its place the moment prefix behavior, ordered iteration, or longest-prefix matching enters the requirements.

**Storing copies of long strings in nodes.** An implementation detail that quietly turns a compressed structure into a memory hog. Store positions and lengths into the original text instead.

## The short version

A trie stores characters in nodes and lets the path define the key, which is why lookup costs exactly the length of the string you're searching for and why prefix queries, wildcard matching, and longest-prefix routing all fall out naturally. The tax is memory: a naive R-way trie allocates R links per node whether you use them or not, and that tax gets brutal fast as the alphabet grows. Compressed tries, ternary search tries, and bitwise variants exist specifically to pay less of it.

If you're studying this for an interview, write the thing from scratch once before you touch a problem list. If you're building autocomplete or a matcher in production, check whether your requirements are actually prefix-shaped, because if they're not, a hash map will do the job with less code and less memory.
