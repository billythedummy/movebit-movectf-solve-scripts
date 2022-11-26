# flash_loan

This challenge involves exploiting a flash loan package that didn't account for how a loan would interact with its other components.

## Writeup (twitter)

This one was pretty easy thanks to move’s concept of abilities https://move-book.com/advanced-topics/types-with-abilities.html?#types-with-abilities

Let’s look at the module. It’s a simple flash loan program. The goal is to steal all its funds.

To take a flash loan, the user calls loan() which transfers him the desired loan amount. The user is then free to perform any other actions, as long as he calls repay() and check() before the end of the transaction to repay the loan.

It’s similar to how @dumbcontract2 ‘s adobe program works https://github.com/2501babe/adobe 

How does the module guarantee that the user repays before the end? This is where move abilities come in. Notice that apart from returning a coin object containing the loaned funds, loan() also returns a “Receipt” struct.

Receipt does not have any of the 4 move abilities. Since it does not has key or store, it is not a move resource and cannot be stored on-chain. So it must be destroyed by the end of the transaction.

Since it does not implement drop, the only way to destroy it is for the module to manually destructure it. The only place where that happens is in check().

So the user must call check() with the created Receipt before the end of the transaction, which verifies that the flash loaned funds have been returned.

So this means we have to call check() somehow in order to destroy the Receipt. Wait a minute, check() just checks the balance. It’s totally decoupled from repay() and both functions are public.

So maybe there’s a way to just call check() without calling repay() and keep the loaned out Coin object...

And indeed there is. What else better to do with the flash loan funds than to provide liquidity to the program so that check() passes and destroys the Receipt for us while crediting us with the pool’s balance?

And that’s it. Add a call to get_flag() after calling expoit() and the flag is ours.

This challenge may seem a little too simple but imo it demonstrates the power of move abilities. Because Receipt doesn’t implement drop, key or store, we focus on the only place where it’s destructured, check().     

Just as how the move abilities informed us of where to look for vulnerabilities, they are able to clearly define how parts of your program work together and how things can go wrong.

While the lack of a guard for disabling deposits and withdraws when a flash loan is active may seem obvious in hindsight, it may not be for more complex programs.

Theoretically any program that controls a pool of  tokens (an AMM for instance) can offer flash loans. However, one must be wary of how they would interact with other parts of the program. 

Imagine an xy=k AMM that offered flash loans from the reserves of either of the pool’s 2 tokens. If there were no guards against trading with the pool when a flash loan is active, and if xy=k was calculated only using the reserve balances, an attacker could easily drain the pool by flash loaning everything out from one side and buying the other for cheap.
