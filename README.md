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

    // Normalize line breaks so parsing is consistent across platforms
    $content = str_replace(["\r\n", "\r"], "\n", $content);

    // Break into individual lines
    $lines = explode("\n", $content);

    // Collect sample lines (first 10 non-blank lines) for delimiter detection
    foreach ($lines as $ln) {
        if (trim($ln) !== "") $debug_info['sample_lines'][] = $ln;
        if (count($debug_info['sample_lines']) >= 10) break;
    }

    /* ------------------------------------------------------------------------
     * STEP 2: AUTO-DETECT CSV DELIMITER
     * ------------------------------------------------------------------------
     * Why this matters:
     *   Some CSV files use commas, others use tabs or semicolons.
     *   Detecting the correct separator avoids broken columns.
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
     * We now parse the entire CSV using the detected delimiter.
     */
    if (($handle = fopen($csv_file, "r")) !== false) {

        // Read header row
        $header_raw = fgets($handle);
        if ($header_raw !== false) {
            $header_raw = strip_bom($header_raw);
            $header = array_map('trim', str_getcsv($header_raw, $best['delim'], '"', "\\"));
            $debug_info['header'] = $header;
        } else {
            $header = [];
        }

        // Read all data rows
        $count = 0;
        while (($line = fgets($handle)) !== false) {

            if (trim($line) === "") continue;
            $cols = str_getcsv($line, $best['delim'], '"', "\\");

            // Map header labels → row values
            $row = [];
            foreach ($header as $i => $colName) {
                $row[$colName] = isset($cols[$i]) ? trim($cols[$i]) : "";
            }

            // Skip rows with no meaningful content
            if (
                ($row['Title'] ?? '') === '' &&
                ($row['Link'] ?? '') === '' &&
                ($row['Reference Link'] ?? '') === ''
            ) continue;

            // Normalize primary fields
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
 * STEP 4: FALLBACK (IF CSV MISSING OR EMPTY)
 * ----------------------------------------------------------------------------
 * This prevents the UI from completely breaking.
 */
if (empty($tiles)) {
    $tiles = [
        ["Title" => "SOPs", "Link" => "#", "Reference Link" => "#", "Description" => "", "Taglines" => "Other", "Icon" => ""],
        ["Title" => "Jira Tickets", "Link" => "#", "Reference Link" => "#", "Description" => "", "Taglines" => "Other", "Icon" => ""],
    ];
}

/* ============================================================================
 * STEP 5: GROUP TILES BY TAGLINES (e.g., Phase 0, Phase 1)
 * ============================================================================ */
$grouped = [];

foreach ($tiles as $t) {

    // Allow multiple tags per tile: "Phase 0, Learning, SOPs"
    $tag_raw = trim($t["Taglines"] ?? "");
    $tags = ($tag_raw === "") ?
            ["Other"] :
            array_filter(array_map('trim', explode(",", $tag_raw)));

    foreach ($tags as $tag) {
        if (!isset($grouped[$tag])) $grouped[$tag] = [];
        $grouped[$tag][] = $t;
    }
}

/**
 * Sort phase groups numerically (Phase 0 → Phase 1 → Other)
 */
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
/* ========================================================================
   UI STYLING — SIMPLE, CLEAN, BRAND MATCHED
   ======================================================================== */
:root{--armi-purple:#3C2375;--armi-dark:#2b1856;--armi-blue:#0071bc;}

body{
    margin:0;
    font-family:'Inter',sans-serif;
    background:var(--armi-purple);
    color:#fff;
}

/* HEADER */
.header{
    background:var(--armi-dark);
    padding:16px 32px;
    display:flex;
    align-items:center;
    gap:18px;
    border-bottom:3px solid rgba(255,255,255,0.12);
}
.header img.logo{height:55px;}
.header-title{font-size:26px;font-weight:600;color:#fff;}

/* MAIN LAYOUT */
.container{padding:28px;max-width:1200px;margin:auto}

/* GRID OF TILES */
.grid{
    display:grid;
    grid-template-columns:repeat(auto-fill,minmax(240px,1fr));
    gap:20px;
}

.tile{
    background:#fff;
    color:var(--armi-dark);
    padding:22px;
    border-radius:12px;
    font-weight:600;
    box-shadow:0 6px 18px rgba(0,0,0,0.28);
    transition:transform .18s,box-shadow .18s;
}
.tile:hover{
    transform:translateY(-6px);
    box-shadow:0 12px 30px rgba(0,0,0,0.36);
}

.tile-icon{
    width:42px;
    height:42px;
    object-fit:contain;
    display:block;
    margin-bottom:12px;
}

.desc{
    font-size:13px;
    color:#444;
    margin-top:8px;
    font-weight:500;
}

.section-title{
    margin-top:32px;
    font-size:22px;
    font-weight:700;
    border-bottom:2px solid rgba(255,255,255,0.25);
    padding-bottom:6px;
}

/* BUTTONS */
.btn-row{
    margin-top:14px;
    display:flex;
    gap:10px;
}

.btn{
    background:var(--armi-blue);
    color:#fff;
    padding:8px 14px;
    border-radius:8px;
    font-size:13px;
    font-weight:600;
    text-decoration:none;
    transition:background .15s,transform .15s;
}

.btn:hover{
    background:#005a96;
    transform:translateY(-2px);
}

.btn.secondary{
    background:#777;
}
.btn.secondary:hover{
    background:#555;
}
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
