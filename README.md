# lowest common ancestor of a binary tree: the one-pass DFS solution, the three cases behind it, and how to stop confusing it with the BST version

The first time you meet this problem, the answer looks like it should be about comparing paths. Find the route from the root to `p`, find the route to `q`, see where they split. That works, and it's a perfectly reasonable way to explain your thinking out loud. It's also not what most interviewers want to see, because it needs two traversals and a data structure holding the routes.

The version they usually want is shorter than the explanation of why it's correct. Six lines of Python, and most of the difficulty lives in one sentence: *a node can be a descendant of itself*.

## The problem, stated without filler

Given a binary tree `root` and two nodes `p` and `q`, return the deepest node that has both `p` and `q` in its subtree. A node counts as its own descendant, which is the detail that decides several test cases.

LeetCode 236 sets the bounds: between 2 and 10⁵ nodes, values in the range of ±10⁹, all values unique, `p != q`, and both `p` and `q` are guaranteed to exist in the tree.

Two examples from the standard tree `[3,5,1,6,2,0,8,null,null,7,4]`:

| `p` | `q` | Answer | Why |
| --- | --- | --- | --- |
| 5 | 1 | 3 | They sit in different subtrees of 3 |
| 5 | 4 | 5 | 4 is inside 5's subtree, so 5 is its own ancestor |

That second row is the one that trips people. Nothing below 5 can be an ancestor of 5, so the answer stops at 5 even though 2 and 4 are deeper.

## The three cases that make the recursion work

Stand on any node in the tree. There are only three places `p` and `q` can be relative to you:

1. Both are somewhere in your left subtree.
2. Both are somewhere in your right subtree.
3. They're split — one on each side — or one of them is you.

In case 3, the node you're standing on is the answer. In cases 1 and 2, the answer is deeper and you keep looking.

Recursion handles this almost for free, because a recursive call on a subtree returns exactly the information you need: "the lowest node in this subtree that matters." If the left call and the right call both come back with something that matters, the current node is where the two targets diverged — so it's the LCA. If only one side comes back non-null, everything relevant is in that side, so you pass that result upward unchanged.

The push-upward part is where people get lost. It helps to think of the return value not as "the answer" but as "the deepest useful thing I found in this subtree."

## The one-pass DFS solution

python
class Solution:
    def lowestCommonAncestor(self, root, p, q):
        if root is None or root is p or root is q:
            return root

        left = self.lowestCommonAncestor(root.left, p, q)
        right = self.lowestCommonAncestor(root.right, p, q)

        if left and right:
            return root
        return left or right


Line by line:

- `root is None` — dead end, report nothing back.
- `root is p or root is q` — you found a target. Return it. This single line is doing double duty: it's both a "found" signal and a candidate answer.
- The two recursive calls explore both subtrees.
- `if left and right` — both sides returned something real, meaning `p` and `q` are on opposite sides of `root`. Return `root`.
- `return left or right` — everything relevant is on one side (or nowhere). Pass it up.

**Why the early return is safe.** If `p` happens to be an ancestor of `q`, you return `p` immediately and never look for `q`. That's correct, because `p` is the LCA by definition — a node is its own descendant. You're not skipping work you needed to do.

**Why you can't short-circuit.** Traversal is postorder: both children finish before the current node decides anything. You have to walk the whole tree. There's no "found it, stop" early exit, which is why this is a backtracking problem rather than a search problem.

Complexity is O(n) time, since each node is visited once, and O(h) space for the recursion stack, where `h` is the tree height. On a skewed tree that's O(n).

## The iterative version, using parent pointers

If you're worried about recursion depth — and on a 10⁵-node skewed tree in Python, with a default recursion limit of 1,000, you should be — the standard iterative answer builds a parent map first.

python
class Solution:
    def lowestCommonAncestor(self, root, p, q):
        parent = {root: None}
        stack = [root]

        while p not in parent or q not in parent:
            node = stack.pop()
            if node.left:
                parent[node.left] = node
                stack.append(node.left)
            if node.right:
                parent[node.right] = node
                stack.append(node.right)

        ancestors = set()
        while p:
            ancestors.add(p)
            p = parent[p]

        while q not in ancestors:
            q = parent[q]

        return q


The logic: walk up from `p`, marking every ancestor including `p` itself. Then walk up from `q` until you hit a node already in that set. The first match is the LCA, because it's the deepest node both paths share.

The traversal stops early once both `p` and `q` have parents recorded, which is a small optimization worth mentioning. Space is O(n) for the parent map — worse than the recursive version's O(h), and a fair trade for not blowing the stack on deep trees.

## The path-comparison approach

Worth knowing, mostly for the subprocedure. Find the path from the root to `p`, find the path to `q`, then walk both arrays until they diverge and return the last common element.

python
def path_to(root, target, path):
    if root is None:
        return False
    path.append(root)
    if root is target:
        return True
    if path_to(root.left, target, path) or path_to(root.right, target, path):
        return True
    path.pop()
    return False


The "path from root to a node" pattern shows up in plenty of other tree problems, so learning it pays off elsewhere. But for LCA specifically, it needs two traversals and O(h) extra storage for the paths, and the recursive solution is shorter. Know it, don't lead with it.

## Mistakes that actually cost you points

**Assuming the answer is always a "split" node.** It isn't. If one target is inside the other's subtree, the answer is the ancestor itself.

**Not noticing what happens when `p` or `q` might be absent.** The one-pass solution returns whatever it found, silently. If the follow-up is "what if only one of them is in the tree," you need the variant that tracks two booleans (`foundP`, `foundQ`) and records the LCA when both flip true at the same node. This is a common follow-up, and it's a different problem.

**Comparing values instead of node identity.** Values are unique in the given constraints, so `root.val == p.val` happens to work here. Node identity is still the safer habit, and it's the only thing that works when values aren't guaranteed unique.

**Recursion depth in Python.** Default limit is 1,000. A 10⁵-node chain will blow it. Mention `sys.setrecursionlimit` or the iterative version before the interviewer asks.

**Handling a single-node tree or the root being one of the targets.** `root = [1,2], p = 1, q = 2` returns 1. Verify your base case covers it.

## Don't file this next to the BST version

LeetCode 235 — "lowest common ancestor of a binary search tree" — looks like the same problem and isn't. The BST's ordering lets you walk down from the root and decide direction in O(h) time with O(1) space, no recursion stack and no postorder propagation:

- If both `p` and `q` are smaller than the current node, go left.
- If both are larger, go right.
- Otherwise the current node is the split point, and that's your answer.

The reason people mix these up: they memorize "go left when both are smaller" and then apply it to a plain binary tree, where left and right mean nothing about ordering. In an unordered tree, both `p` and `q` being in the left subtree tells you nothing until you've recursed. If your answer to 236 involves a while loop comparing values, you've solved 235.

## Follow-ups you should have answers for

- **N-ary tree.** Same postorder logic, just iterate over `node.children` instead of two fixed children.
- **LCA of K nodes.** Count how many of the targets appear in each subtree instead of tracking two flags.
- **With parent pointers and no root.** Walk both nodes up to equal depth, then advance them together until they meet. O(h) space, no stack.
- **Multiple queries on a static tree.** Precompute with binary lifting or Euler tour plus sparse table for O(log n) per query. Rare in interviews, common in competitive programming.
- **LCA in a DAG.** Different beast. A DAG can have several minimal common ancestors, so "the" LCA isn't well defined without extra rules. Say that out loud rather than writing code.

## Where structured practice helps

The reason LCA feels slippery isn't the code — it's the reasoning about return values propagating upward. That's a pattern, not a trick, and patterns are exactly what AlgoMonster organizes its curriculum around. It was built by ex-Google engineers and competitive programmers, and the whole pitch is that interview problems reuse a small set of approaches, so you learn the approach instead of memorizing solutions.

Concretely, for this problem, several things on the platform map onto it directly:

- **Tree recursion and postorder return-value propagation** — the section LCA belongs to. The core skill is deciding what your recursive function returns and how the parent combines the children's results.
- **Decision flowcharts** — AlgoMonster publishes a flowchart that walks problem descriptions down to an algorithm family, so "which pattern is this?" becomes a procedure instead of a guess. It's designed as a reference, not something to memorize.
- **Code templates** — for a problem like this, where the bug is usually a wrong combination step rather than a syntax error, having a known-good return-value skeleton to work from saves real time.
- **Inline IDE and algorithm visualizer** — you write and run the code on the page, and can step through pointers and recursion with animations. For backtracking problems, watching the call stack unwind is more useful than reading about it.
- **Speedrun mode** — skim problems rather than writing full solutions each time, which is how you cover enough trees and graphs problems to actually retain the pattern.
- **AI explanations** — select any passage and get it re-explained at your level. Useful when a written explanation of return-value propagation doesn't land the first time.

Solutions are available across Python, Java, JavaScript, C++, Go, and Haskell. The content is text plus interactive diagrams — there's no video at all, which is a genuine mismatch if you learn by watching someone else code. The problem catalogue is also much smaller than LeetCode's 3,000+, and system design coverage is introductory rather than senior-level. The site currently advertises 500+ lessons across 6 courses, 1,000+ illustrations, and 3,000+ editorials, plus two included bonus courses (a CS foundation course and a build-an-LLM course), and a member discount on their live bootcamps.

You can look through the tree-recursion material and the flowchart before paying anything: 👉 [explore AlgoMonster's curriculum and current pricing](https://algo.monster?fpr=li69).

## Plans and what they cost

AlgoMonster shows a free tier plus two paid plans. There's no monthly billing option on the subscribe page — it's annual or one-time.

| Plan | What you get | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of patterns and lessons — enough to judge the teaching style, not enough to prepare seriously | $0 | Free | [start with the free tier](https://algo.monster?fpr=li69) |
| Pro – Yearly | All patterns, lessons, and problems unlocked; unlimited AI assistant; unlimited Speedrun; company questions updated monthly; 500+ lessons across 6 courses; 1,000+ illustrations; 3,000+ editorials; member discount on live bootcamps; 12 months of access | $69.99/year (listed as $5.84/mo) | Billed annually, renews | [get the Pro Yearly plan](https://algo.monster?fpr=li69) |
| Pro – Lifetime | Everything in the annual plan, plus lifetime updates — new content and future courses included, no renewal | $89 | One-time | [get the Pro Lifetime plan](https://algo.monster?fpr=li69) |

A caveat you should factor in rather than ignore: this platform runs sales constantly, and the numbers on the subscribe page move a lot. During research the same page showed annual pricing at $69.99/year and lifetime at $89, while other recent listings showed $119/year and $459 lifetime, and third-party reviews have cited figures like $99/year and $189. Check the checkout page for the number you'd actually be charged — the plans and what's included are stable, the prices aren't.

For comparison, third-party breakdowns put LeetCode Premium around $159/year and NeetCode around $119/year or $219 lifetime. AlgoMonster's lifetime tier is usually the cheapest of the three, though LeetCode's problem bank is roughly an order of magnitude larger.

## Who the lifetime plan is actually for

The math is straightforward once you're honest about your situation. If you have an interview in six weeks, $89 or $69.99 for one year covers you and renewing would be wasted money. If you expect to interview again in two or three years — promotions, job changes, the usual — the lifetime plan costs less than two annual renewals at these prices, and it's the one that doesn't lapse while you're not looking.

What you shouldn't buy it for: video walkthroughs (there are none), mock interviews with a human (not offered), or senior-level system design (too shallow). If any of those is your actual gap, spend the money elsewhere and use AlgoMonster for the DSA portion only.

## Quick answers

**Is the recursive solution the expected answer?** Usually, yes. It's O(n) time and O(h) space with one traversal, and it demonstrates that you understand postorder return values. Being able to also write the parent-pointer version is a plus, not a requirement.

**Why does `return left or right` work?** Because at most one of the two can be non-null when you reach that line — if both were non-null you'd have already returned `root` above. `left or right` is just a compact way of picking whichever subtree contained the targets.

**Does the algorithm need both nodes to exist?** Yes. The one-pass version assumes it and will return an incorrect answer if one target is missing. Say this out loud in an interview; it's the kind of constraint awareness that gets noticed.

**Do I need the path-comparison version too?** No, but the "find path from root to node" subroutine is worth knowing on its own, since it reappears in problems about printing paths, checking ancestry, and reconstructing trees.

**Is 235 or 236 more likely to come up?** Both show up in tree rounds. If the problem statement says "binary search tree," the O(h)/O(1) ordered walk is the answer, and writing the general recursion instead signals that you didn't read the constraint.
