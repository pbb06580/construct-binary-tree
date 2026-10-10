# Construct Binary Tree from Preorder and Inorder Traversal: The O(n) Hash Map Solution, Traced Through LeetCode 105

You get two arrays. `preorder` is the preorder traversal of a binary tree, `inorder` is the inorder traversal of the same tree, and you have to give back the tree itself. That's LeetCode 105, and it shows up in interview loops often enough that "can you rebuild a tree from traversals" is worth more than a quick skim.

The standard example:


preorder = [3, 9, 20, 15, 7]
inorder  = [9, 3, 15, 20, 7]

Returns:
      3
     / \
    9   20
       /  \
      15   7


Constraints from the problem: up to 3,000 nodes, values can range from -3000 to 3000, all values are **unique**, and both arrays are guaranteed to describe a real tree.

That uniqueness constraint is doing a lot of work. Remove it and the whole approach changes.

## The two facts everything rests on

Before writing a line of code, be able to say these out loud:

1. **The first element of `preorder` is the root.** Preorder is root → left → right, so the very first value you record is the top of the tree. Always. No exceptions.
2. **Everything to the left of the root in `inorder` belongs to the left subtree; everything to the right belongs to the right subtree.** Inorder is left → root → right, so the root's position in that array is a physical divider.

Those two facts are the entire algorithm. Everything else is bookkeeping.

So for the example: `preorder[0]` is `3`, so root = 3. Where does 3 sit in `inorder`? At index 1. That means `[9]` is the left subtree and `[15, 20, 7]` is the right subtree.

Now the problem has shrunk. You need to build a tree from `preorder = [9]`, `inorder = [9]` — trivial — and another from `preorder = [20, 15, 7]`, `inorder = [15, 20, 7]`. Repeat until the subproblems are empty.

## The baseline: O(n²), and why you should still write it

The obvious version scans `inorder` linearly to find each root's position:

python
def buildTree(preorder, inorder):
    if not preorder:
        return None

    root_val = preorder[0]
    root_index = inorder.index(root_val)   # O(n) scan, every call

    node = TreeNode(root_val)
    node.left  = buildTree(preorder[1:root_index + 1], inorder[:root_index])
    node.right = buildTree(preorder[root_index + 1:], inorder[root_index + 1:])
    return node


This is correct, and it's a reasonable thing to say first in an interview. It's O(n²) in the worst case because every recursive call scans the inorder array, and a skewed tree gives you O(n) levels. The slicing also copies arrays, which is extra memory work most people forget to mention.

Then you say: "the scan is the bottleneck, so let's make root lookup O(1)."

## The O(n) version: index maps instead of slicing

Trade the slicing for three integers per call — where the subtree starts in `preorder`, where it starts in `inorder`, and how many nodes it holds.

python
def buildTree(preorder, inorder):
    index_of = {value: i for i, value in enumerate(inorder)}

    def build(pre_start, in_start, size):
        if size <= 0:
            return None

        root_val = preorder[pre_start]
        root_in  = index_of[root_val]          # O(1) instead of a scan
        left_size = root_in - in_start

        node = TreeNode(root_val)
        node.left  = build(pre_start + 1, in_start, left_size)
        node.right = build(pre_start + 1 + left_size, root_in + 1, size - 1 - left_size)
        return node

    return build(0, 0, len(preorder))


Hash map build is O(n) once. Every recursive call does constant work plus two recursive calls, one per node. Total: **O(n) time, O(n) space** for the map, plus O(h) recursion depth — O(log n) on a balanced tree, O(n) if the tree is a linked list wearing a tree costume.

## The index arithmetic is where people actually lose points

The recurrence itself is easy to memorize. Deriving the two child index pairs from scratch, under pressure, in front of someone watching, is the part that fails. Walk through it slowly on the example.

First call: `pre_start = 0`, `in_start = 0`, `size = 5`. Root is `preorder[0] = 3`. In `inorder = [9, 3, 15, 20, 7]`, 3 is at index 1 (`root_in = 1`). Left subtree size = `1 - 0 = 1`.

- **Left child call:** in `preorder`, the left subtree starts right after the root, so `pre_start + 1 = 1`. In `inorder`, it starts where the current segment starts, so `in_start` stays 0. Size is `left_size = 1`. That's `build(1, 0, 1)` → node 9, no children.
- **Right child call:** in `preorder`, skip the root and all `left_size` nodes of the left subtree: `pre_start + 1 + left_size = 2`. In `inorder`, start after the root: `root_in + 1 = 2`. Size is everything except the root and the left subtree: `size - 1 - left_size = 3`. That's `build(2, 2, 3)`.

That third call handles `preorder = [20, 15, 7]` and `inorder = [15, 20, 7]`: root 20, `root_in = 3`, `left_size = 3 - 2 = 1`, left child is `build(3, 2, 1)` → 15, right child is `build(4, 4, 1)` → 7. Tree matches the expected output.

The equivalent variant uses explicit bounds instead of a subtree size — a single index pointer walking `preorder` forward, plus `inorder` low/high limits:

python
def buildTree(preorder, inorder):
    index_of = {value: i for i, value in enumerate(inorder)}
    pre_index = 0

    def build(lo, hi):
        nonlocal pre_index
        if lo > hi:
            return None
        root_val = preorder[pre_index]
        pre_index += 1
        node = TreeNode(root_val)
        mid = index_of[root_val]
        node.left  = build(lo, mid - 1)
        node.right = build(mid + 1, hi)
        return node

    return build(0, len(inorder) - 1)


Fewer parameters to get wrong, but it depends on `pre_index` advancing exactly once per node — which only holds because the traversal order matches. If you mutate that pointer in the wrong branch, the test fails on the second recursive case and you get to spend ten minutes finding it.

Pick one formulation, trace it on paper against the example above, and keep it in your head. Half-following both is how you end up with subtrees attached to the wrong parents.

## Edge cases worth naming before your interviewer does

- **Empty input.** Not possible per the constraints (`preorder.length >= 1`), but `size <= 0` returning `None` is what terminates every recursion, so it has to be there.
- **Single node.** `preorder = [-1]`, `inorder = [-1]` → a root with two `None` children. A nice sanity check that your base case is on the right variable.
- **Completely skewed trees.** 3,000 nodes all on the left chain. Time stays O(n), but recursion depth hits 3,000 — fine in Python at the default limit, a real problem in languages with small stacks. Worth mentioning that an explicit stack version exists.
- **Duplicate values.** The problem promises unique values specifically so the index map is well defined. With duplicates you can't tell which occurrence of a value is the root, and the reconstruction is ambiguous. If asked, the honest answer is: you need the duplicates broken by position, not just value.

## The follow-up questions this problem invites

**"What if I give you inorder and postorder instead?"** That's LeetCode 106. Same shape, inverted: the last element of `postorder` is the root, and the index map is built from `inorder` again. If you can derive 105 from first principles, 106 is a five-minute adjustment.

**"What about preorder and postorder?"** This one is a trap, and it's worth answering slowly. Two traversals aren't automatically enough. For a two-node tree, a root `a` with a single left child `b` gives `preorder = ab`, `postorder = ba`. A root `a` with a single right child `b` gives the same two arrays. Same input, two different trees. Preorder and postorder don't uniquely define a binary tree — that's why 105 uses inorder, which carries the left/right boundary information that the other two don't.

**"The tree is a BST, and I only give you preorder."** Then you can rebuild it, because the sorted version of the values gives you the inorder array for free, and you're back to 105 with `sorted(preorder)` as the second input.

**"Can you do it iteratively with a stack?"** Yes — push nodes down the left chain, pop when the next preorder value matches the top of the stack, then attach the remaining values as right children. It's a real technique but a worse answer under time pressure unless you've drilled it.

The pattern underneath all of these is the same: recognize that the traversal order tells you where the root is, split the problem at that root, recurse on both halves, and keep the index bookkeeping straight.

## Where the walkthrough lives, and what's free

The version of this solution on AlgoMonster sits at `algo.monster/liteproblems/105`, alongside per-problem editorials for the numbered LeetCode set — TypeScript, Python, Java, and C++ variants included. Those liteproblems pages are readable without a paid plan, which makes them a reasonable first stop when you want a second explanation of one specific problem.

👉 [Read the AlgoMonster 105 walkthrough and the rest of the liteproblems set](https://algo.monster/liteproblems/105?fpr=li69)

Where the paid side matters is the pattern layer. Tree reconstruction is one application of DFS with index bookkeeping — the same reasoning that handles subtree sums, path problems, and serialization. AlgoMonster packages that as an explicit pattern with a template and a decision flowchart, rather than leaving you to notice the connection across forty unrelated problems. The platform was built by ex-Google engineers, is text-and-diagram based rather than video-based, and its full subscribe page lists 500+ lessons across 6 courses, 1,000+ illustrations, and 3,000+ editorials, plus AI assistant usage and Speedrun mode for timed drilling.

## AlgoMonster plans, what they cost, and what's behind the paywall

Pricing is where review sites go sideways. AlgoMonster runs rotating sales, so third-party posts quote figures from different sale windows — $99/year and $189 lifetime in some, $459 lifetime in others, "$99–150" in the comparison tables. The numbers below come from AlgoMonster's own subscribe page at the time of writing; the checkout screen is the only place that shows the final charge.

| Plan | What you get | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of patterns and problems, free account, liteproblems editorials | $0 | — | [Create a free AlgoMonster account](https://algo.monster?fpr=li69) |
| Pro – Yearly | All patterns, lessons and problems; unlimited AI assistant; unlimited Speedrun; company question banks updated monthly; 500+ lessons, 1,000+ illustrations, 3,000+ editorials; member discount on live bootcamps; bonus Foundation and Build-an-LLM courses | $5.84/mo, billed annually at $69.99/year | 12 months, cancel or upgrade anytime | [See the AlgoMonster yearly plan](https://algo.monster/subscribe?fpr=li69) |
| Pro – Lifetime | Everything in the yearly plan, plus lifetime updates and all future content at no extra cost | $89 | One-time payment | [Check the AlgoMonster lifetime plan](https://algo.monster/subscribe?fpr=li69) |

There's no monthly plan and no enterprise tier — it's the free tier, one annual option, one one-time option. If your interview is six weeks out, the yearly plan covers you and costs less than a single month of most competing subscriptions. If you expect to interview again in two or three years, the lifetime option at $89 is priced close enough to a single year that the decision mostly answers itself.

The caveats third-party reviews consistently raise are real, so factor them in: the curriculum is **text and diagrams only, no videos**, which is either the reason it's fast or the reason it doesn't stick, depending on how you learn. Problem volume is around 231 mapped problems against LeetCode's 3,000+, because the product is the pattern framework, not the problem bank. System design coverage is introductory. And you can finish the whole curriculum in roughly 4–8 weeks, so if you're buying lifetime access, be honest about whether you'll actually come back to it.

## FAQ

**Is there a working discount code?** Affiliate and coupon sites advertise codes that stack on top of the current sale, but codes expire and vary by campaign. Trust what the checkout shows after a code is applied — not a blog post's table from six months ago.

**Does the free tier cover LeetCode 105?** The liteproblems editorial for 105 is accessible without paying. The pattern module that teaches the underlying approach, the templates, and Speedrun drilling sit behind the subscription.

**Can I solve 105 without a hash map?** Yes, by scanning `inorder` for each root — correct, just O(n²). With the 3,000-node constraint, that still passes on a modern judge for many inputs, but the O(n) version is what the follow-up question is fishing for.

**Which language should I practice it in?** Whichever you'll interview in. The index arithmetic is identical everywhere; TypeScript's `Map`, Java's `HashMap`, and Python's dict are the same idea. AlgoMonster's templates ship in several languages, which makes side-by-side comparison easy if you're switching between them.

**Why does my solution return the wrong tree structure?** Almost always the right-subtree preorder start index. It's `pre_start + 1 + left_size`, not `pre_start + left_size` — and the size parameter is `size - 1 - left_size`, not `size - left_size`. Those two off-by-ones account for most failed submissions on this problem.
