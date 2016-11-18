---
published: true
title: Matching IPv6 addresses against CIDR masks in MySQL
layout: post
tags: [mysql, ipv6, dev]
---

CIDR matching is a common task operating large sets of IP addresses. As more networks are getting IPv6 support, we start receiving IPv6 datasets for analysis and need to match those against CIDR masks as one of steps.

Doing so in MySQL is impossible in the most common way - by applying bitmasks. Binary representation of IPv6 addresses are 128-bit numbers, while longest integer type in MySQL is `BIGINT`, 8 bytes (64 bits) long. That's why result of `INET6_ATON()` is `VARBINARY`, and you can't have MySQL apply bitwise operators on it.

Another approach is to calculate first and last IPv6 addresses matching given CIDR mask, and match those entries falling between the two. This is discussed [on stackoverflow](http://stackoverflow.com/q/16611717), but the proposed implementation is buggy. Here's the proper solution we coded for our project:

```sql
DELIMITER $$

CREATE FUNCTION `FirstIPv6MatchingCIDR` (`ip` VARCHAR(46), `mask` INT(2) UNSIGNED) RETURNS VARCHAR(39) DETERMINISTIC
    BEGIN
        RETURN INET6_NTOA(UNHEX(RPAD(
            CONCAT(
                SUBSTR(HEX(INET6_ATON(`ip`)), 1, FLOOR(`mask` / 4)),
                HEX(
                    ((POW(2, `mask` % 4) - 1) * POW(2, 4 - (`mask` % 4)))
                    & COALESCE(CONV(SUBSTR(HEX(INET6_ATON(`ip`)), FLOOR(`mask` / 4) + 1, 1), 16, 10), '')
                )
            ),
            32, 0
        )));
    END$$

CREATE FUNCTION `LastIPv6MatchingCIDR` (`ip` VARCHAR(46), `mask` INT(2) UNSIGNED) RETURNS VARCHAR(39) DETERMINISTIC
    BEGIN
        DECLARE `ipNumber` VARBINARY(16);
        DECLARE `last` VARCHAR(39) DEFAULT '';
        DECLARE `flexBits`, `counter`, `deci`, `newByte` INT UNSIGNED;
        DECLARE `hexIP` VARCHAR(32);

        SET `ipNumber` = INET6_ATON(`ip`);
        SET `hexIP`    = HEX(`ipNumber`);
        SET `flexBits` = 128 - `mask`;
        SET `counter`  = 32;

        WHILE (`flexBits` > 0) DO
            SET `deci`    = CONV(SUBSTR(`hexIP`, `counter`, 1), 16, 10);
            SET `newByte` = `deci` | (POW(2, LEAST(4, `flexBits`)) - 1);
            SET `last`    = CONCAT(CONV(`newByte`, 10, 16), `last`);

            IF `flexBits` >= 4 THEN
                SET `flexBits` = `flexBits` - 4;
            ELSE
                SET `flexBits` = 0;
            END IF;

            SET `counter`  = `counter` - 1;
        END WHILE;

        SET `last` = CONCAT(SUBSTR(`hexIP`, 1, `counter`), `last`);

        RETURN INET6_NTOA(UNHEX(`last`));
    END $$

DELIMITER ;

```

Example `SELECT`:

```sql
SELECT @firstIP := INET6_ATON(FirstIPv6MatchingCIDR('::ffff:0:0', 96));
SELECT @lastIP := INET6_ATON(LastIPv6MatchingCIDR('::ffff:0:0', 96));

SELECT `ip` FROM `ip_db` WHERE INET6_ATON(`ip`) BETWEEN @firstIP AND @lastIP;
```
