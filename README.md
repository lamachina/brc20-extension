# 🪙 Universal BRC-20 Extension Tracker

This repository provides a simple and effective way to **index and analyze Universal BRC-20 token deployments and minting activity** using [Dune.com](https://dune.com).

## 🔍 Purpose

The goal is to enable seamless tracking of BRC-20 `deploy` and `mint` operations through a SQL-based query that can be executed directly on Dune Analytics. This helps researchers, developers, and users to gain insights into token supply, minting progress, and activity trends in the BRC-20 ecosystem.

## 📊 How It Works

This query parses `OP_RETURN` data in Bitcoin transactions to extract and decode embedded BRC-20 JSON payloads. It then isolates and validates `deploy` and `mint` operations, aggregates totals, and presents a comprehensive overview of each token's state.

## ✅ What It Tracks

- **Token Ticker**
- **Total Supply**
- **Mint Limit**
- **Deployment Time & TXID**
- **Total Amount Minted**
- **Minting Progress (% of supply)**

## 🛠 How to Use

1. Go to [Dune.com](https://dune.com)
2. Create a new query
3. Paste in the full SQL code below
4. Configure the following parameters:

| Parameter         | Description                          | Example           |
|------------------|--------------------------------------|-------------------|
| `start_date`      | Start date for scanning transactions | `2025-05-01`      |
| `end_date`        | End date (exclusive)                 | `2025-05-11`      |
| `max_json_bytes`  | Maximum JSON payload size to parse   | `80`              |

5. Run the query to explore indexed BRC-20 data!

## 📦 Example Output

| Ticker | Total Minted | Total Supply | Mint Limit | Deployment Time | TXID  | Minted % |
|--------|--------------|--------------|------------|-----------------|-------|----------|
| ABCD   | 1,000        | 10,000       | 100        | 2025-05-02       | ...   | 10.0%    |

## 📄 SQL Query

```sql

WITH
  op_return_payload_parsing AS (
    SELECT
      block_time,
      tx_id, 
      TRY_CAST(script_hex AS VARCHAR) AS script_hex_varchar,
      SUBSTRING(TRY_CAST(script_hex AS VARCHAR), 5, 2) AS first_byte_after_op_return_hex 
    FROM bitcoin.outputs
    WHERE
      SUBSTRING(TRY_CAST(script_hex AS VARCHAR), 3, 2) = '6a' 
      AND block_time >= DATE(TRY_CAST('2025-05-01' AS TIMESTAMP))
      AND block_time < DATE(TRY_CAST('2025-05-12' AS TIMESTAMP))
  ),
  extracted_raw_payload_hex AS (
    SELECT
      block_time, tx_id, script_hex_varchar, first_byte_after_op_return_hex,
      LOWER( 
        CASE
          WHEN first_byte_after_op_return_hex = '4d' THEN SUBSTRING(script_hex_varchar, 11) 
          WHEN first_byte_after_op_return_hex = '4c' THEN SUBSTRING(script_hex_varchar, 9) 
          WHEN TRY(CAST(from_base(first_byte_after_op_return_hex, 16) AS INTEGER)) > 0 
           AND TRY(CAST(from_base(first_byte_after_op_return_hex, 16) AS INTEGER)) < 76 THEN 
            SUBSTRING(CAST(bytearray_substring(FROM_HEX(SUBSTRING(script_hex_varchar, 5)),2) AS VARCHAR ), 3) 
          ELSE NULL 
        END
      ) AS raw_payload_hex 
    FROM op_return_payload_parsing
  ),
  decoded_raw_payload_text AS (
    SELECT
      tx_id, block_time, raw_payload_hex,
      TRY(from_utf8(FROM_HEX(raw_payload_hex))) AS full_decoded_text,
      COALESCE(LENGTH(raw_payload_hex) / 2, 0) AS initial_payload_byte_length
    FROM extracted_raw_payload_hex
    WHERE raw_payload_hex IS NOT NULL AND raw_payload_hex <> ''
  ),
  isolated_json_details AS ( 
    SELECT
      tx_id, block_time, full_decoded_text, initial_payload_byte_length,
      CASE
        WHEN STRPOS(full_decoded_text, '{') > 0 AND STRPOS(SUBSTRING(full_decoded_text, STRPOS(full_decoded_text, '{')), '}') > 0 THEN
          SUBSTRING(full_decoded_text, STRPOS(full_decoded_text, '{'), STRPOS(SUBSTRING(full_decoded_text, STRPOS(full_decoded_text, '{')), '}'))
        ELSE NULL
      END AS potential_json_substring,
      COALESCE(LENGTH(TO_HEX(TRY_CAST( (CASE
                                WHEN STRPOS(full_decoded_text, '{') > 0 AND STRPOS(SUBSTRING(full_decoded_text, STRPOS(full_decoded_text, '{')), '}') > 0 THEN
                                  SUBSTRING(full_decoded_text, STRPOS(full_decoded_text, '{'), STRPOS(SUBSTRING(full_decoded_text, STRPOS(full_decoded_text, '{')), '}'))
                                ELSE NULL
                              END) AS VARBINARY))) / 2, 0) AS isolated_json_byte_length
    FROM decoded_raw_payload_text
    WHERE full_decoded_text IS NOT NULL
  ),
  parsed_and_filtered_json AS (
    SELECT
      tx_id, block_time, potential_json_substring,
      TRY(json_parse(potential_json_substring)) AS parsed_json_object
    FROM isolated_json_details
    WHERE 
      potential_json_substring IS NOT NULL
      AND isolated_json_byte_length <= CAST('80' AS INTEGER) 
      AND isolated_json_byte_length > 0 
  ),
  brc20_mints AS (
    SELECT
      LOWER(TRY_CAST(json_extract_scalar(parsed_json_object, '$.tick') AS VARCHAR)) AS ticker_normalized,
      TRY_CAST(TRY_CAST(json_extract_scalar(parsed_json_object, '$.amt') AS VARCHAR) AS DECIMAL(38,0)) AS amount_minted
    FROM parsed_and_filtered_json
    WHERE
      parsed_json_object IS NOT NULL
      AND LOWER(TRY_CAST(json_extract_scalar(parsed_json_object, '$.p') AS VARCHAR)) = 'brc-20'
      AND LOWER(TRY_CAST(json_extract_scalar(parsed_json_object, '$.op') AS VARCHAR)) = 'mint'
      AND TRY_CAST(TRY_CAST(json_extract_scalar(parsed_json_object, '$.amt') AS VARCHAR) AS DECIMAL(38,0)) IS NOT NULL
  ),
  brc20_deploys AS (
    SELECT
      LOWER(TRY_CAST(json_extract_scalar(parsed_json_object, '$.tick') AS VARCHAR)) AS ticker_normalized,
      TRY_CAST(TRY_CAST(COALESCE(json_extract_scalar(parsed_json_object, '$.max'), json_extract_scalar(parsed_json_object, '$.m')) AS VARCHAR) AS DECIMAL(38,0)) AS total_supply,
      TRY_CAST(TRY_CAST(COALESCE(json_extract_scalar(parsed_json_object, '$.lim'), json_extract_scalar(parsed_json_object, '$.l')) AS VARCHAR) AS DECIMAL(38,0)) AS mint_limit,
      block_time, 
      tx_id
    FROM parsed_and_filtered_json
    WHERE 
      parsed_json_object IS NOT NULL
      AND LOWER(TRY_CAST(json_extract_scalar(parsed_json_object, '$.p') AS VARCHAR)) = 'brc-20'
      AND LOWER(TRY_CAST(json_extract_scalar(parsed_json_object, '$.op') AS VARCHAR)) IN ('deploy', 'd')
      AND TRY_CAST(TRY_CAST(COALESCE(json_extract_scalar(parsed_json_object, '$.max'), json_extract_scalar(parsed_json_object, '$.m')) AS VARCHAR) AS DECIMAL(38,0)) IS NOT NULL
      AND TRY_CAST(TRY_CAST(COALESCE(json_extract_scalar(parsed_json_object, '$.lim'), json_extract_scalar(parsed_json_object, '$.l')) AS VARCHAR) AS DECIMAL(38,0)) IS NOT NULL
  ),
  aggregated_mints AS (
    SELECT
      ticker_normalized,
      SUM(amount_minted) AS total_amount_minted
    FROM brc20_mints
    WHERE ticker_normalized IS NOT NULL AND amount_minted IS NOT NULL
    GROUP BY 1
  ),
  unique_deploys AS (
    SELECT
      ticker_normalized,
      total_supply,
      mint_limit, 
      block_time AS deploy_block_time, 
      TO_HEX(tx_id) AS deploy_tx_id, 
      rn
    FROM (
      SELECT
        ticker_normalized,
        total_supply,
        mint_limit, 
        block_time, 
        tx_id,    
        ROW_NUMBER() OVER (PARTITION BY ticker_normalized ORDER BY block_time ASC) as rn 
      FROM brc20_deploys
      WHERE ticker_normalized IS NOT NULL AND total_supply IS NOT NULL 
    ) subquery_for_unique_deploys
    WHERE rn = 1
  )
SELECT
  d.ticker_normalized AS ticker,
  COALESCE(m.total_amount_minted, 0) AS total_amount_minted,
  d.total_supply,
  d.mint_limit, 
  d.deploy_block_time, 
  d.deploy_tx_id, 
  CASE
    WHEN d.total_supply > 0 THEN ROUND((CAST(COALESCE(m.total_amount_minted, 0) AS DOUBLE) / CAST(d.total_supply AS DOUBLE)) * 100, 2)
    ELSE 0 
  END AS percentage_minted
FROM unique_deploys d 
LEFT JOIN aggregated_mints m ON d.ticker_normalized = m.ticker_normalized 
WHERE
  d.ticker_normalized IS NOT NULL 
ORDER BY
  percentage_minted ASC, d.total_supply DESC, d.ticker_normalized ASC;
```


# 🪙 BRC-20 Mint Tracker for a Target Address

This query provides an approximate way to analyze BRC-20 `mint` inscriptions received by a **specific Bitcoin address**, based on transaction structure and `OP_RETURN` payloads.

> ⚠️ This is **not a complete BRC-20 balance tracker**.

## ✅ What This Tracks

- BRC-20 `mint` operations only
- Associates the mint with the **first standard output** of the transaction
- Sums the total `amt` received per ticker at the given address

## 🛠 Parameters

| Parameter           | Description                                      | Example         |
|---------------------|--------------------------------------------------|-----------------|
| `start_date`         | Starting date for OP_RETURN parsing             | `'2025-05-01'`  |
| `end_date`           | End date (exclusive)                            | `'2025-05-11'`  |
| `max_json_bytes`     | Max JSON size to parse (OP_RETURN byte length)  | `80`            |
| `target_address`     | Bitcoin address to filter for                   | `bc1...`        |

## 📄 SQL Query

```sql

WITH
  op_return_base AS (
    SELECT
      tx_id,
      block_time,
      index AS op_return_output_index,
      TRY_CAST(script_hex AS VARCHAR) AS script_hex_varchar
    FROM bitcoin.outputs
    WHERE
      type = 'nulldata'
      AND SUBSTRING(TRY_CAST(script_hex AS VARCHAR), 3, 2) = '6a'
      AND block_time >= DATE(TRY_CAST('{{start_date}}' AS TIMESTAMP))
      AND block_time < DATE(TRY_CAST('{{end_date}}' AS TIMESTAMP))
  ),
  extracted_raw_payload AS (
    SELECT
      tx_id,
      block_time,
      LOWER(
        CASE
          WHEN SUBSTRING(script_hex_varchar, 5, 2) = '4d' THEN SUBSTRING(script_hex_varchar, 11)
          WHEN SUBSTRING(script_hex_varchar, 5, 2) = '4c' THEN SUBSTRING(script_hex_varchar, 9)
          WHEN TRY(CAST(from_base(SUBSTRING(script_hex_varchar, 5, 2), 16) AS INTEGER)) BETWEEN 1 AND 75 THEN
            SUBSTRING(CAST(bytearray_substring(FROM_HEX(SUBSTRING(script_hex_varchar, 5)), 2) AS VARCHAR), 3)
          ELSE NULL
        END
      ) AS raw_payload_hex
    FROM op_return_base
  ),
  decoded_payload AS (
    SELECT
      tx_id,
      block_time,
      raw_payload_hex,
      TRY(from_utf8(FROM_HEX(raw_payload_hex))) AS full_decoded_text
    FROM extracted_raw_payload
    WHERE raw_payload_hex IS NOT NULL AND raw_payload_hex <> ''
  ),
  isolated_json AS (
    SELECT
      tx_id,
      block_time,
      full_decoded_text,
      CASE
        WHEN STRPOS(full_decoded_text, '{') > 0 AND STRPOS(SUBSTRING(full_decoded_text, STRPOS(full_decoded_text, '{')), '}') > 0 THEN
          SUBSTRING(
            full_decoded_text,
            STRPOS(full_decoded_text, '{'),
            STRPOS(SUBSTRING(full_decoded_text, STRPOS(full_decoded_text, '{')), '}')
          )
        ELSE NULL
      END AS potential_json_substring
    FROM decoded_payload
    WHERE full_decoded_text IS NOT NULL
  ),
  parsed_and_filtered_json AS (
    SELECT
      tx_id,
      block_time,
      TRY(json_parse(potential_json_substring)) AS json_data
    FROM isolated_json
    WHERE 
      potential_json_substring IS NOT NULL
      AND COALESCE(LENGTH(TO_HEX(TRY_CAST(potential_json_substring AS VARBINARY))) / 2, 0) <= CAST('{{max_json_bytes}}' AS INTEGER)
      AND COALESCE(LENGTH(TO_HEX(TRY_CAST(potential_json_substring AS VARBINARY))) / 2, 0) > 0
  ),
  brc20_mint_inscriptions AS (
    SELECT
      tx_id,
      block_time,
      LOWER(TRY_CAST(json_extract_scalar(json_data, '$.tick') AS VARCHAR)) AS ticker_normalized,
      TRY_CAST(TRY_CAST(json_extract_scalar(json_data, '$.amt') AS VARCHAR) AS DECIMAL(38,0)) AS amount_minted
    FROM parsed_and_filtered_json
    WHERE 
      json_data IS NOT NULL 
      AND LOWER(TRY_CAST(json_extract_scalar(json_data, '$.p') AS VARCHAR)) = 'brc-20'
      AND LOWER(TRY_CAST(json_extract_scalar(json_data, '$.op') AS VARCHAR)) = 'mint'
      AND TRY_CAST(json_extract_scalar(json_data, '$.tick') AS VARCHAR) IS NOT NULL 
      AND TRY_CAST(TRY_CAST(json_extract_scalar(json_data, '$.amt') AS VARCHAR) AS DECIMAL(38,0)) IS NOT NULL 
  ),
  first_non_op_return_outputs AS (
    SELECT
      o.tx_id,
      o.address AS recipient_address,
      ROW_NUMBER() OVER (PARTITION BY o.tx_id ORDER BY o.index ASC) AS rn
    FROM bitcoin.outputs o
    INNER JOIN (SELECT DISTINCT tx_id FROM brc20_mint_inscriptions) relevant_mints 
      ON o.tx_id = relevant_mints.tx_id
    WHERE 
      o.type <> 'nulldata'
      AND o.address IS NOT NULL AND o.address <> ''
  ),
  mints_to_target_address AS (
    SELECT
      m.ticker_normalized,
      m.amount_minted
    FROM brc20_mint_inscriptions m
    JOIN first_non_op_return_outputs f 
      ON m.tx_id = f.tx_id AND f.rn = 1
    WHERE LOWER(f.recipient_address) = LOWER('{{target_address}}')
  )
SELECT
  ticker_normalized AS ticker,
  SUM(amount_minted) AS total_balance_from_mints
FROM mints_to_target_address
WHERE ticker_normalized IS NOT NULL AND amount_minted IS NOT NULL
GROUP BY 1
ORDER BY total_balance_from_mints DESC, ticker;
