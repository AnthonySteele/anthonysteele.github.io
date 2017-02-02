Structured logging

## need for robustness

loggoer.LogException() is typically called when things are going wrong.

The logging code has more exposure to edge cases than regular code. If your logging code fails, now you have two problems.

Your logging failing is like if your ambulance crashes.


## appropriate logging strategies

vary by app scale
- email
- shared sql
- own SQL
- ELMAH
- ELK

Structure becomes important at scale, to sort through lots of data.



