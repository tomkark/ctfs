Another netcating challenge.

It would not be fun for me to copy the full netcat output and try make it displayable so let's just get right into the fun parts.

```bash
Cash: $500
-------
|C    |
|  K  |
|    C|
-------

Your Total is 10

The Dealer Has a Total of 10

Enter Bet: $
```

Just a cute blackjack game. You bet X cash: you win, you double it, you lose, you lose ;)

But according to the challenge's description the flag should be only given to millionaires... So how can we become millionaries without doing many games?

Let's try the trivial option, just bet way more money than we have:
```bash
Cash: $500
-------
|C    |
|  K  |
|    C|
-------

Your Total is 10

The Dealer Has a Total of 10

Enter Bet: $10000

You cannot bet more money than you have.
Enter Bet:
```

Let's find this string in the source code in the description:

```c
int betting() //Asks user amount to bet
{
 printf("\n\nEnter Bet: $");
 scanf("%d", &bet);
 
 if (bet > cash) //If player tries to bet more money than player has
 {
        printf("\nYou cannot bet more money than you have.");
        printf("\nEnter Bet: ");
        scanf("%d", &bet);
        return bet;
 }
 else return bet;
 ```

 Uhm, looks like it doesn't loop over the same condition, so apparently we can just try to bet a lot of money in the second time and it should work.

```bash
Cash: $500
-------
|C    |
|  K  |
|    C|
-------

Your Total is 10

The Dealer Has a Total of 10

Enter Bet: $10000

You cannot bet more money than you have.
Enter Bet: 10000000000


Would You Like to Hit or Stay?
Please Enter H to Hit or S to Stay.
```

Ok so we just need to win this single game.

```bash
Cash: $500
-------
|C    |
|  K  |
|    C|
-------

Your Total is 10

The Dealer Has a Total of 10

Enter Bet: $10000

You cannot bet more money than you have.
Enter Bet: 10000000000


Would You Like to Hit or Stay?
Please Enter H to Hit or S to Stay.
H
-------
|S    |
|  K  |
|    S|
-------

Your Total is 20

The Dealer Has a Total of 20

Would You Like to Hit or Stay?
Please Enter H to Hit or S to Stay.
S

You Have Chosen to Stay at 20. Wise Decision!

The Dealer Has a Total of 20
Unbelievable! You Win!

You have 1 Wins and 0 Losses. Awesome!

Would You Like To Play Again?
Please Enter Y for Yes or N for No
```

And after entering 'Y':
```bash
YaY_**********************


Cash: $1410065908
-------
|D    |
|  K  |
|    D|
-------

Your Total is 10

The Dealer Has a Total of 10

Enter Bet: $
```

