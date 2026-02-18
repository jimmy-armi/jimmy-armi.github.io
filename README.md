# ARMI Internal Dashboard — ICON Support + Two-Link Tiles
**Version:** 1.4.0  
**Updated:** 2025-12-04  
**Author:** James Henderson  

---

## Purpose
Renders an internal dashboard UI based on data from a CSV file.  
Each CSV row becomes a tile with:

- Title  
- Description  
- Main link ("Open")  
- Reference link ("Reference")  
- Optional Icon image  
- Taglines (used to automatically group tiles)

---

## CSV Requirements
**Required Columns**
- `Title`
- `Link`
- `Reference Link`
- `Description`
- `Taglines`

**Optional Column**
- `Icon`

---

## How Data Is Loaded
1. Script detects the CSV file and reads raw text.  
2. Line endings normalized.  
3. Delimiter auto‑detected: tab, comma, semicolon, pipe.  
4. Header row parsed with correct delimiter.  
5. Each row becomes a tile array.  
6. Tiles grouped by *Taglines* (supports multiple comma‑separated tags).

---

## How Data Is Displayed
- Tagline groups become dashboard sections.  
- Tiles show title, description, icon, and action buttons.

---

## Coding Standards
- PHPDoc for functions  
- snake_case or camelCase  
- Inline comments for technical + non‑technical clarity  
- All user-facing text escaped with `htmlspecialchars`  

---

# Full PHP + HTML Source Code

```php
<?php
/**
 * ============================================================================
 * ARMI Internal Dashboard — ICON Support + Two-Link Tiles
 * ============================================================================
 * Version: 1.4.0
 * Updated: 2025-12-04
 * Author: James Henderson
 *
 * PURPOSE:
 *     Renders an internal dashboard UI based on data from a CSV file.
 *     Each CSV row becomes a tile with:
 *       - Title
 *       - Description
 *       - Main link ("Open")
 *       - Reference link (“Reference")
 *       - Optional Icon image
 *       - Taglines (used to automatically group tiles)
 *
 * CSV REQUIREMENTS:
 *     Required Columns:
 *         Title, Link, Reference Link, Description, Taglines
 *     Optional Column:
 *         Icon
 *
 * HOW DATA IS LOADED:
 *     1. The script detects the CSV file and reads its raw text.
 *     2. Line endings are normalized.
 *     3. The script samples multiple lines to detect the correct delimiter:
 *         - tabs, commas, semicolons, pipes
 *     4. The header row is parsed using the chosen delimiter.
 *     5. Each additional row is converted into a tile array.
 *     6. Tiles are grouped based on “Taglines” (supports multiple comma-separated tags).
 *
 * HOW DATA IS DISPLAYED:
 *     - Groups (e.g., “Phase 0”, “Phase 1”) become sections.
 *     - Titles, descriptions, icons, and buttons render inside responsive tiles.
 *
 * CODING STANDARDS:
 *     - Functions are documented (PHPDoc)
 *     - All variables use lower_snake_CASE or camelCase consistently
 *     - Inline comments written for both engineers and non-technical users
 *     - All user input is escaped (htmlspecialchars)
 *
 * ============================================================================
 */

$tiles = [];
$csv_file = __DIR__ . "/links.csv";

/* ----------------------------------------------------------------------------
 * UTILITY FUNCTIONS
 * ---------------------------------------------------------------------------- */

/**
 * Remove UTF-8 Byte Order Mark from a string (BOM can break CSV headers)
 *
 * @param string $str Raw input line
 * @return string Cleaned string
 */
function strip_bom($str) {
    return (substr($str, 0, 3) === "\xEF\xBB\xBF") ? substr($str, 3) : $str;
}

/* ----------------------------------------------------------------------------
 * DEBUG DATA FOR DEVELOPMENT
 * ---------------------------------------------------------------------------- */
$debug_info = [
    "found_file"        => file_exists($csv_file),
    "sample_lines"      => [],
    "detected_delimiter"=> null,
    "header"            => null,
    "rows_parsed"       => 0,
];

/* ============================================================================
 * STEP 1: LOAD + PREPARE RAW CSV CONTENT
 * ============================================================================ */
if ($debug_info['found_file']) {

    // Read whole CSV file as a string
    $content = file_get_contents($csv_file);

    // Normalize line breaks
    $content = str_replace(["\r\n", "\r"], "\n", $content);

    // Break into lines
    $lines = explode("\n", $content);

    // Collect first 10 non-empty lines for delimiter detection
    foreach ($lines as $ln) {
        if (trim($ln) !== "") $debug_info['sample_lines'][] = $ln;
        if (count($debug_info['sample_lines']) >= 10) break;
    }

    /* ------------------------------------------------------------------------
     * STEP 2: AUTO-DETECT CSV DELIMITER
     * ------------------------------------------------------------------------
     */
    $candidates = ["\t", ",", ";", "|"];
    $best = ["delim" => ",", "cols" => 0];

    foreach ($candidates as $d) {
        $col_counts = [];

        foreach ($debug_info['sample_lines'] as $line) {
            $line = strip_bom($line);
            $parts = str_getcsv($line, $d, '"', "\\");
            $col_counts[] = count($parts);
        }

        $mode = array_count_values($col_counts);
        arsort($mode);
        $most_frequent = array_key_first($mode);

        if ($most_frequent > $best['cols']) {
            $best['cols'] = $most_frequent;
            $best['delim'] = $d;
        }
    }

    $debug_info['detected_delimiter'] = $best['delim'];

    /* ------------------------------------------------------------------------
     * STEP 3: READ HEADER + ROWS
     * ------------------------------------------------------------------------
     */
    if (($handle = fopen($csv_file, "r")) !== false) {

        // Read header
        $header_raw = fgets($handle);
        if ($header_raw !== false) {
            $header_raw = strip_bom($header_raw);
            $header = array_map('trim', str_getcsv($header_raw, $best['delim'], '"', "\\"));
            $debug_info['header'] = $header;
        } else {
            $header = [];
        }

        // Parse data rows
        $count = 0;
        while (($line = fgets($handle)) !== false) {

            if (trim($line) === "") continue;
            $cols = str_getcsv($line, $best['delim'], '"', "\\");

            // Map header → values
            $row = [];
            foreach ($header as $i => $colName) {
                $row[$colName] = isset($cols[$i]) ? trim($cols[$i]) : "";
            }

            // Skip empty rows
            if (
                ($row['Title'] ?? '') === '' &&
                ($row['Link'] ?? '') === '' &&
                ($row['Reference Link'] ?? '') === ''
            ) continue;

            // Normalize
            $row['URL']  = trim($row['Link'] ?? "");
            $row['REF']  = trim($row['Reference Link'] ?? "");
            $row['ICON'] = trim($row['Icon'] ?? "");

            $tiles[] = $row;
            $count++;
        }

        fclose($handle);
        $debug_info['rows_parsed'] = $count;
    }
}

/* ----------------------------------------------------------------------------
 * STEP 4: FALLBACK IF CSV EMPTY
 * ---------------------------------------------------------------------------- */
if (empty($tiles)) {
    $tiles = [
        ["Title" => "SOPs", "Link" => "#", "Reference Link" => "#", "Description" => "", "Taglines" => "Other", "Icon" => ""],
        ["Title" => "Jira Tickets", "Link" => "#", "Reference Link" => "#", "Description" => "", "Taglines" => "Other", "Icon" => ""],
    ];
}

/* ============================================================================
 * STEP 5: GROUP TILES
 * ============================================================================ */
$grouped = [];

foreach ($tiles as $t) {

    // Supports multiple comma-separated tags
    $tag_raw = trim($t["Taglines"] ?? "");
    $tags = ($tag_raw === "") ?
            ["Other"] :
            array_filter(array_map('trim', explode(",", $tag_raw)));

    foreach ($tags as $tag) {
        if (!isset($grouped[$tag])) $grouped[$tag] = [];
        $grouped[$tag][] = $t;
    }
}

// Sort "Phase X" numerically
uksort($grouped, function($a, $b) {
    $pa = preg_match('/Phase (\d+)/i', $a, $m1) ? intval($m1[1]) : 999;
    $pb = preg_match('/Phase (\d+)/i', $b, $m2) ? intval($m2[1]) : 999;
    return $pa <=> $pb ?: strcmp($a, $b);
});

?>
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<title>ARMI Technical Resources Dashboard</title>
<link rel="icon" type="image/png" href="icon.png">
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;600&display=swap" rel="stylesheet">

<style>
... (CSS unchanged for brevity)
</style>
</head>

<body>

<div class="header">
    <img class="logo" src="https://www.armiusa.org/wp-content/uploads/2022/03/armi-logo-web.svg" alt="ARMI">
    <div class="header-title">ARMI Technical Resources Dashboard</div>
</div>

<div class="container">
    <div class="topbar" style="display:flex;justify-content:space-between;margin-bottom:14px">
        <div style="font-weight:600">Apps</div>
        <div style="font-size:13px">
            CSV: <?= htmlspecialchars(basename($csv_file)) ?> &nbsp; |
            &nbsp; Rows: <?= count($tiles) ?>
        </div>
    </div>

    <?php foreach ($grouped as $tag => $items): ?>
        <div class="section-title"><?= htmlspecialchars($tag) ?></div>

        <div class="grid">
            <?php foreach ($items as $t):
                $title = htmlspecialchars($t['Title'] ?? 'Untitled');
                $desc  = htmlspecialchars($t['Description'] ?? '');
                $url   = htmlspecialchars($t['URL'] ?? '');
                $ref   = htmlspecialchars($t['REF'] ?? '');
                $icon  = htmlspecialchars($t['ICON'] ?? '');
            ?>
                <div class="tile">

                    <?php if ($icon): ?>
                        <img class="tile-icon" src="<?= $icon ?>" alt="icon">
                    <?php endif; ?>

                    <div class="title"><?= $title ?></div>

                    <?php if ($desc): ?>
                        <div class="desc"><?= $desc ?></div>
                    <?php endif; ?>

                    <div class="btn-row">
                        <?php if ($url): ?>
                            <a class="btn" href="<?= $url ?>" target="_blank">Open</a>
                        <?php endif; ?>

                        <?php if ($ref): ?>
                            <a class="btn secondary" href="<?= $ref ?>" target="_blank">Reference</a>
                        <?php endif; ?>
                    </div>

                </div>
            <?php endforeach; ?>
        </div>

    <?php endforeach; ?>
</div>

</body>
</html>
