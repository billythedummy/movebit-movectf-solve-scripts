# simple_game

This challenge involves recreating on-chain pseudo-RNG in order to preview these generated variables, thereby giving yourself an unfair advantage playing the on-chain game.

## Writeup (twitter)

This is one of the most interesting challenges I’ve done (in my limited experience of 2 CTFs).

At first glance, it’s a simple RPG game where you control a “Hero” to fight 2 different types of monsters – “Boar” and “BoarKing”. The result of the fight is determined by the stats – hp, strength, defense, of the Hero and the monster.

Your Hero gains experience as you win fights, and can level up once for a huge boost to stats.

At the core of the game is a a `rand_u64_range()` function. It generates the RNG for determining monster stats.

It also determines if a slain BoarKing drops a “TreasuryBox”. You get the flag when you defeat the BoarKing and consume the TreasuryBox it drops to call the get_flag() function.

Furthermore, the flag is only given if the rand_u64_range() fun returns 0, otherwise your hard-earned TreasuryBox will be consumed for nothing.

There’s also a stamina stat that limits the number of fights you can take to either 100 boar kings or 200 boars, so that limits our number of attempts.

With the odds stacked against us, this means we will need to somehow exploit the RNG mechanism to allow us to “win” every time before our stamina runs out.

Taking a look at the rand_u64_range() def, the core of it is a seed() fun that generates some random-looking bytes, which rand_u64_range() then takes the first 8 bytes of as a u64 and then modulos it to return a number in the range provided by the caller.

Let’s look at seed(). So the random-looking bytes are from sha3_256(bcs::to_bytes(ctx) || object::uid_to_bytes(object::new(ctx)), where || denotes concatenation.

Wtf’s a TxContext? bcs::to_bytes()? object::new()? object::uid_to_bytes()? I didn’t know yet, but all this seems like data that’s globally accessible by any executing code, which is fatal if your RNG uses them as input.

Why? This means users can recreate the RNG logic to preview any generated variables and abort the transaction if the RNG isn’t in their favour. Which means they only ever win or pay the (small) transaction fees for failed transactions without losing.

As a side note, this pattern of attack can be used for any lottery-esque game on-chain: if a user can determine within the transaction whether he’s going to win or lose the lottery, then he can simply check and abort transactions where he loses.

This is probably a big reason why @degencoinflip games are split into 2 transactions, a GoDegen instruction for users to commit their choice of head or tails,
and then a FlipForDegenerate instruction that’s signed by their servers to actually perform the RNG and determine the result. There’s no way for the user to know if he will win or lose from the first transaction.

Anyway, back to the exploit. So, all we have to do is figure out how to recreate the RNG, then create a wrapper contract that recreates the RNG and calls the game’s various methods only if RNG is in our favour, and the flag is ours. Let’s start digging into the sui code.

After some googling, I managed to find this important folder: https://github.com/MystenLabs/sui/tree/main/crates/sui-framework/docs

First off, lets figure out what bcs::to_bytes() does. It’s serialization, just as its name suggests. The summary says that it’s little-endian and packed, just like borsh. Cool, that’s about all I need. https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/docs/bcs.md#module-0x2bcs

What about TxContext? As suspected, it’s a globally accessible struct that’s passed into any entrypoint function. So that should cover the first half of our sha3_256() input, right? https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/docs/tx_context.md#struct-txcontext

Nope, looks like object::new() presents a curveball. TxContext has a counter that keeps track of how many new objects have been created in the transaction, and increments with every object::new() call. This means ctx mutates so we cant just copy the code and use bcs::to_bytes(ctx).
https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/docs/tx_context.md#function-new_object

Wait, but what’s input to the sha3_256() is just the bcs serialized bytes of the TxContext. We can totally just spoof a fake ids_created. Looking at the struct defn of TxContext, it seems like the last 8 bytes of its bcs repr is ids_created. https://github.com/MystenLabs/sui/blob/7424aba44aece90f2969171629773bdc6c79ed7e/crates/sui-framework/sources/tx_context.move#L25-L35

Note: am lazy in my impl here and just modify the first byte since I knew ids_created < 256. Also, random side-note: TxContext has a bcs serialized length of 69. Nice.

The bcs::to_bytes(ctx) drops another clue. For the second half of the sha3_256() input, it seems like we need ctx.tx_hash, but that’s a private variable since move only allows the defining module to access struct fields directly and there’s no public getter function. https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/docs/tx_context.md#function-new_object

Well, we don’t need a getter function if we can get its raw bytes via bcs. bytes[length-16-32: length-16] should be it.

And finally for the second input to derive_id() we can just pass in any desired ids_created. Got lazy here again, only modify first byte yadee yadah.

Now, we can recreate any RNG at any point in the game as long as know how many times ids_created was incremented in the code.

Let’s go ahead and kill some boars without losing. For this, I created a winnable() function that checks if your hero can win a fight, and wrapped the various slay_*() functions with a check that aborts if !winnable(). Screenshot of winnable() and slay_boar()

Deploy the package (I called it “esg” with module name “e”) and spam away with the sui cli. The nice thing about the cli is that it exits with code 0 if the transaction is successful and nonzero if it’s not, so you can just use an until in bash to spam until successful.

After killing 100 boars and using exactly 100 stamina, I could level up my hero for the stats boost.

Then, it was time to kill the boar king. This one needs an additional check to see if the boar king will drop a TreasuryBox.

After 20 minutes of spamming the sui devnet, I finally got the treasury box. Another 20 minutes of spamming gf() and the flag was emitted.

Ah right, omitted a pretty important detail: i also had to dig through the sui src code for derive_id to figure out that its actually sha3_256: https://github.com/MystenLabs/sui/blob/fe1db4b50425c28693a34564bd8b54be8a68ad89/crates/sui-framework/src/natives/tx_context.rs#L16 , https://github.com/MystenLabs/sui/blob/fe1db4b50425c28693a34564bd8b54be8a68ad89/crates/sui-types/src/base_types.rs#L441
