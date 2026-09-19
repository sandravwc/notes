---
title: cheat sheet/clickhouse
---

- query rspamd mail log by symbol/domain/sender
  - ```sql
    SELECT RcptDomain, Score, Action, TS, symbol, From, symbolscore, option, Subject
    FROM ( SELECT RcptDomain, Score, Action, Date, MessageId, From, Subject, TS,
           Symbols.Names AS symbols, Symbols.Scores AS symbolscores, Symbols.Options AS options
           FROM rspamd )
    ARRAY JOIN symbols AS symbol, symbolscores AS symbolscore, options AS option
    WHERE (Date = '2025-04-02') AND match(symbol, 'SPF') AND match(RcptDomain, 'example') AND match(From, 'sender-domain')
    ORDER BY TS ASC
    -- Date: year-month-day. symbol/RcptDomain/From take regex, e.g. '(SPF|DMARC_DKIM)_INVALIDATIONS'
    ```
