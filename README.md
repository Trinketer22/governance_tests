# Governance contracts tests

This project provides tests for [elector](https://github.com/ton-blockchain/ton/blob/master/crypto/smartcont/elector-code.fc) and [config](https://github.com/ton-blockchain/ton/blob/master/crypto/smartcont/config-code.fc) contracts.  
Each directory contains tests implemented with [toncli](https://github.com/disintar/toncli) testing framework and written testing plan according to which those were implemented.

## General info

High lever description of both contracts can be found in [Governance contracts](https://ton.org/docs/develop/smart-contracts/governance) manual.

## Toncli related info

Please follow [Toncli installation](https://github.com/disintar/toncli/blob/master/INSTALLATION.md) manual.  
I'd personally recommend [Docker](https://hub.docker.com/r/trinketer22/func_docker/) as a fastest to date way of getting started with toncli on Linux/MacOS/Windows.

### Possible gotchas

#### Docker
If while running pre-built docker image you get `SIGILL` (Illegal instruction) exception, please report issue to the [Docker repo](https://hub.docker.com/r/trinketer22/func_docke).  
In this case you're going to have to build an image from scratch following the [build section](https://github.com/Trinketer22/func_docker#build) of the docker manual.

#### Pre-built binaries

If you've decided to go with the pre-built binaries instead, these are the latest builds:

- [Linux](https://github.com/SpyCheese/ton/suites/10688806393/artifacts/535014992)
- [MacOS](https://github.com/SpyCheese/ton/suites/9535611669/artifacts/453290127)
- [Windows](https://github.com/Trinketer22/ton-main/suites/10995875839/artifacts/557516778) 

Vigilant reader might notice that **Windows** link points to a different repo.  
That's because windows artifacts had expired in the main repo.


If any of those expire, all you have to do is to fork [SpyCheese](https://github.com/SpyCheese/ton) repo, and run build action yourself.  
Don't forget to pick toncli-local branch prior to action execution.

#### Self build

Please follow official [compilation how-to](https://ton.org/docs/develop/howto/compile).  
Note that for `toncli` to function you only need [FunC](https://ton.org/docs/develop/howto/compile#func), [Fift](https://ton.org/docs/develop/howto/compile#fift) and [Lite Client](https://ton.org/docs/develop/howto/compile#lite-client) sections.

## General workflow

If all tools are set, one can just `cd` to the selected contract directory and run:
`toncli run_tests` or in case of *Docker* follow:[Run tests](https://github.com/Trinketer22/governance_tests.git) section of manual.

Note that full set execution might take some time.  
One can select specific test cases or make use of asterisk(`*`) mask to run only matching test cases.  
Like for example in config_tests:

``` bash
toncli run_tests proposal_change_pubkey_int vote_proposal_accepted
```

Will pick only specified tests.  
Note that `__test_` prefix in test case name may be omitted.

In case one wants to specify cases by mask:

``` bash
toncli run_tests *vote_ext*
```
