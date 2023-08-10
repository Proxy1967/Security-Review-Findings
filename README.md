# Security-Review-Findings
| Platform and Contest | Severity | Name | Picked for Report |
| :---: | :----: | :----------------: | :--------------: |
| [Code4rena Frankencoin](https://code4rena.com/reports/2023-04-frankencoin) | Medium| Reorg attack on position creation can be used to steal funds | NO |
| [Sherlock Footium](https://github.com/sherlock-audit/2023-04-footium-judging/issues) | Medium | Use SafeERC20 functions where applicable | NO |
| [Sherlock USSD](https://github.com/sherlock-audit/2023-05-USSD-judging/issues) | High | Not using slippage parameter or deadline while swapping on UniswapV3 | YES |
| [Sherlock USSD](https://github.com/sherlock-audit/2023-05-USSD-judging/issues) | High | Precision loss can cause protocol to sell no collateral and leaving it unable to rebalance to peg | NO |
| [Sherlock USSD](https://github.com/sherlock-audit/2023-05-USSD-judging/issues) | High | Oracle will return too expensive price of DAI because of excessive precision scaling | NO |
| [Sherlock USSD](https://github.com/sherlock-audit/2023-05-USSD-judging/issues) | High | Oracle address not setup | NO |
| [Sherlock USSD](https://github.com/sherlock-audit/2023-05-USSD-judging/issues) | High | Using wrong oracle address will cause the protocol to buy wrong amount of collateral and mint wrong amount of USSD token | NO |
| [Sherlock USSD](https://github.com/sherlock-audit/2023-05-USSD-judging/issues) | Medium | Chainlink oracle return values are not handled properly | NO |
