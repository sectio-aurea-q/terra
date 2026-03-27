// ╔══════════════════════════════════════════════════════════════════════════╗
// ║  MEG-APSU TERRA v0.5.0 — Quantum Biodiversity Scanner (PARALLEL)       ║
// ║  sectio-aurea-q · MEGALODON Research · 2026                             ║
// ║                                                                          ║
// ║  "What quantum machinery dies when this species dies?"                   ║
// ║                                                                          ║
// ║  Usage: terra [species_file.txt] [--limit N] [--threads T]               ║
// ╚══════════════════════════════════════════════════════════════════════════╝

use std::env;
use std::fs;
use std::io::{Write, BufRead, BufReader};
use std::thread;
use std::time::{Duration, Instant};
use std::process::Command;
use std::collections::HashSet;
use std::sync::{Arc, Mutex};

// ═══ RESULT ═════════════════════════════════════════════════════════════

#[derive(Clone)]
#[derive(Debug)]
struct SpeciesResult {
    idx: usize,
    scientific_name: String,
    status: String,
    n_enzymes: usize,
    n_pdb: usize,
    n_qc: usize,
    skipped: bool,
}

// ═══ CURL HELPER ════════════════════════════════════════════════════════

fn curl_json(url: &str) -> Option<serde_json::Value> {
    let output = Command::new("curl")
        .args(&["-s", "--connect-timeout", "10", "--max-time", "30", url])
        .output().ok()?;
    serde_json::from_slice(&output.stdout).ok()
}

// ═══ UNIPROT ════════════════════════════════════════════════════════════

fn uniprot_get_enzymes(organism: &str) -> Vec<(String, String, Vec<String>)> {
    let org = organism.replace(' ', "+");
    let url = format!(
        "https://rest.uniprot.org/uniprotkb/search?query=organism_name:%22{}%22+AND+ec:*&format=json&size=50&fields=accession,protein_name,xref_pdb,ec",
        org
    );
    let body = match curl_json(&url) { Some(b) => b, None => return vec![] };
    let results = match body.get("results").and_then(|r| r.as_array()) { Some(r) => r, None => return vec![] };

    let mut enzymes: Vec<(String, String, Vec<String>)> = vec![];
    let mut seen_ec: HashSet<String> = HashSet::new();

    for entry in results {
        let acc = entry.get("primaryAccession").and_then(|v| v.as_str()).unwrap_or("").to_string();
        let name = entry.get("proteinDescription")
            .and_then(|pd| pd.get("recommendedName"))
            .and_then(|rn| rn.get("fullName"))
            .and_then(|fn_| fn_.get("value"))
            .and_then(|v| v.as_str()).unwrap_or("Unknown").to_string();
        let ec = entry.get("proteinDescription")
            .and_then(|pd| pd.get("recommendedName"))
            .and_then(|rn| rn.get("ecNumbers"))
            .and_then(|ec| ec.as_array())
            .and_then(|arr| arr.first())
            .and_then(|e| e.get("value"))
            .and_then(|v| v.as_str()).unwrap_or("").to_string();
        let mut pdbs: Vec<String> = vec![];
        if let Some(xrefs) = entry.get("uniProtKBCrossReferences").and_then(|x| x.as_array()) {
            for xref in xrefs {
                if xref.get("database").and_then(|d| d.as_str()) == Some("PDB") {
                    if let Some(id) = xref.get("id").and_then(|v| v.as_str()) { pdbs.push(id.to_string()); }
                }
            }
        }
        if pdbs.is_empty() && !ec.is_empty() && !seen_ec.contains(&ec) {
            seen_ec.insert(ec.clone());
            if let Some(hp) = find_human_ortholog_pdb(&ec) { pdbs = hp; }
        }
        if !acc.is_empty() { enzymes.push((acc, name, pdbs)); }
    }
    enzymes
}

fn find_human_ortholog_pdb(ec: &str) -> Option<Vec<String>> {
    let url = format!(
        "https://rest.uniprot.org/uniprotkb/search?query=organism_id:9606+AND+ec:{}+AND+database:pdb+AND+reviewed:true&format=json&size=1&fields=accession,xref_pdb",
        ec
    );
    let body = curl_json(&url)?;
    let entry = body.get("results")?.as_array()?.first()?;
    let mut pdbs: Vec<String> = vec![];
    if let Some(xrefs) = entry.get("uniProtKBCrossReferences").and_then(|x| x.as_array()) {
        for xref in xrefs {
            if xref.get("database").and_then(|d| d.as_str()) == Some("PDB") {
                if let Some(id) = xref.get("id").and_then(|v| v.as_str()) { pdbs.push(id.to_string()); break; }
            }
        }
    }
    if pdbs.is_empty() { None } else { Some(pdbs) }
}

// ═══ PDB + MEG-APSU ════════════════════════════════════════════════════

fn download_pdb(pdb_id: &str, cache: &str) -> bool {
    let path = format!("{}/{}.pdb", cache, pdb_id.to_lowercase());
    if std::path::Path::new(&path).exists() { return true; }
    let url = format!("https://files.rcsb.org/download/{}.pdb", pdb_id.to_uppercase());
    let output = match Command::new("curl").args(&["-s", "--connect-timeout", "10", "--max-time", "30", "-o", &path, &url]).output() {
        Ok(o) => o, Err(_) => return false,
    };
    output.status.success()
}

fn scan_with_meg_apsu(pdb_path: &str, meg_binary: &str) -> Option<(f64, bool)> {
    let output = Command::new(meg_binary).args(&["scan", pdb_path, "--json"]).output().ok()?;
    let json: serde_json::Value = serde_json::from_slice(&output.stdout).ok()?;
    let qvs = json.get("qvs").and_then(|v| v.as_f64())?;
    let crit = json.get("class").and_then(|v| v.as_str())
        .map(|c| c.contains("QUANTUM-CRITICAL")).unwrap_or(false);
    Some((qvs, crit))
}

// ═══ SCAN ONE SPECIES ═══════════════════════════════════════════════════

fn scan_one(idx: usize, genus: &str, species: &str, cache: &str, meg: &str) -> SpeciesResult {
    let org = format!("{} {}", genus, species);

    // Skip IUCN — we already know they're CR from the file
    let enzymes = uniprot_get_enzymes(&org);
    let n_enz = enzymes.len();

    if n_enz == 0 {
        return SpeciesResult { idx, scientific_name: org, status: "CR".into(), n_enzymes: 0, n_pdb: 0, n_qc: 0, skipped: false };
    }

    let mut n_pdb = 0usize;
    let mut n_qc = 0usize;

    for (_acc, _name, pdbs) in &enzymes {
        if let Some(pdb_id) = pdbs.first() {
            if download_pdb(pdb_id, cache) {
                n_pdb += 1;
                let pdb_path = format!("{}/{}.pdb", cache, pdb_id.to_lowercase());
                if let Some((_score, critical)) = scan_with_meg_apsu(&pdb_path, meg) {
                    if critical { n_qc += 1; }
                }
            }
        }
    }

    SpeciesResult { idx, scientific_name: org, status: "CR".into(), n_enzymes: n_enz, n_pdb, n_qc, skipped: false }
}

// ═══ SPECIES FILE READER ════════════════════════════════════════════════

fn read_species_file(path: &str) -> Vec<(String, String)> {
    let file = match fs::File::open(path) {
        Ok(f) => f, Err(e) => { eprintln!("  Cannot open {}: {}", path, e); return vec![]; }
    };
    BufReader::new(file).lines().filter_map(|l| {
        let l = l.ok()?.trim().to_string();
        if l.is_empty() { return None; }
        let parts: Vec<&str> = l.splitn(2, ' ').collect();
        if parts.len() == 2 { Some((parts[0].to_string(), parts[1].to_string())) } else { None }
    }).collect()
}

// ═══ MAIN ════════════════════════════════════════════════════════════════

fn main() {
    let start = Instant::now();

    eprintln!("\x1b[36m\x1b[1m");
    eprintln!("  ╔══════════════════════════════════════════════════════════════╗");
    eprintln!("  ║  MEG-APSU TERRA v0.5.0 — Quantum Biodiversity Scanner      ║");
    eprintln!("  ║  PARALLEL MODE — sectio-aurea-q · 2026                      ║");
    eprintln!("  ║                                                              ║");
    eprintln!("  ║  \"What quantum machinery dies when this species dies?\"       ║");
    eprintln!("  ╚══════════════════════════════════════════════════════════════╝");
    eprintln!("\x1b[0m");

    let args: Vec<String> = env::args().collect();
    let species_file = args.get(1).map(|s| s.as_str()).unwrap_or("cr_species_500.txt");
    let mut limit: usize = 100;
    let mut n_threads: usize = 8;
    for i in 0..args.len() {
        if args[i] == "--limit" { if let Some(n) = args.get(i+1) { limit = n.parse().unwrap_or(100); } }
        if args[i] == "--threads" { if let Some(n) = args.get(i+1) { n_threads = n.parse().unwrap_or(8); } }
        if args[i] == "--all" { limit = 999999; }
    }

    let home = env::var("HOME").unwrap_or("/tmp".into());
    let cache = format!("{}/.terra-pdb-cache", home);
    let _ = fs::create_dir_all(&cache);
    let meg = format!("{}/meg-apsu/target/release/meg-apsu", home);
    if !std::path::Path::new(&meg).exists() {
        eprintln!("  \x1b[31mMEG-APSU not found\x1b[0m"); std::process::exit(1);
    }

    let all_species = read_species_file(species_file);
    if all_species.is_empty() {
        eprintln!("  \x1b[31mNo species in {}\x1b[0m", species_file); std::process::exit(1);
    }
    let scan_count = std::cmp::min(limit, all_species.len());

    eprintln!("  {} species loaded, scanning {} with {} threads",
        all_species.len(), scan_count, n_threads);
    eprintln!("  Pipeline: UniProt → PDB → Lindblad (IUCN skipped — all CR)");
    eprintln!("  Cache: {}\n", cache);

    // Build work queue
    let work: Vec<(usize, String, String)> = all_species.iter().take(scan_count)
        .enumerate().map(|(i, (g, s))| (i, g.clone(), s.clone())).collect();

    let work = Arc::new(Mutex::new(work.into_iter().collect::<std::collections::VecDeque<_>>()));
    let results = Arc::new(Mutex::new(Vec::<SpeciesResult>::new()));
    let counter = Arc::new(Mutex::new(0usize));

    // Spawn threads
    let mut handles = vec![];
    for _t in 0..n_threads {
        let work = Arc::clone(&work);
        let results = Arc::clone(&results);
        let counter = Arc::clone(&counter);
        let cache = cache.clone();
        let meg = meg.clone();
        let total = scan_count;

        handles.push(thread::spawn(move || {
            loop {
                let item = { work.lock().unwrap().pop_front() };
                let (idx, genus, species) = match item { Some(i) => i, None => break };

                let result = scan_one(idx, &genus, &species, &cache, &meg);

                let mut cnt = counter.lock().unwrap();
                *cnt += 1;
                let n = *cnt;
                drop(cnt);

                let org = format!("{} {}", genus, species);
                if result.n_enzymes == 0 {
                    eprintln!("  [{:3}/{}] {:<35} → no enzymes", n, total, org);
                } else {
                    eprintln!("  [{:3}/{}] {:<35} → {:3} enz {:3} PDB \x1b[{}m{:2} QC\x1b[0m",
                        n, total, org, result.n_enzymes, result.n_pdb,
                        if result.n_qc > 0 { "33;1" } else { "32" }, result.n_qc);
                }

                results.lock().unwrap().push(result);

                // Small delay to avoid API hammering
                thread::sleep(Duration::from_millis(200));
            }
        }));
    }

    for h in handles { h.join().unwrap(); }

    let elapsed = start.elapsed();

    // Sort results by index
    let mut results = Arc::try_unwrap(results).unwrap().into_inner().unwrap();
    results.sort_by_key(|r| r.idx);

    // ═══ SUMMARY ═══
    let n = results.len();
    let t_enz: usize = results.iter().map(|r| r.n_enzymes).sum();
    let t_pdb: usize = results.iter().map(|r| r.n_pdb).sum();
    let t_qc: usize = results.iter().map(|r| r.n_qc).sum();
    let s_qc = results.iter().filter(|r| r.n_qc > 0).count();
    let s_with_enz = results.iter().filter(|r| r.n_enzymes > 0).count();

    eprintln!("\n  \x1b[36m\x1b[1m═══ TERRA v0.5.0 SCAN COMPLETE ═══\x1b[0m");
    eprintln!("  \x1b[2mTime: {:.1}s ({:.1} species/min)\x1b[0m\n",
        elapsed.as_secs_f64(), n as f64 / (elapsed.as_secs_f64() / 60.0));

    eprintln!("  Species scanned:         {}", n);
    eprintln!("  With enzymes in UniProt: {}", s_with_enz);
    eprintln!("  Total enzymes:           {}", t_enz);
    eprintln!("  With PDB structure:      {}", t_pdb);
    eprintln!("  Quantum-critical:        {}", t_qc);
    eprintln!("  Species with QC enzymes: {}/{}", s_qc, s_with_enz);
    if t_pdb > 0 { eprintln!("  QC rate:                 {:.1}%", (t_qc as f64 / t_pdb as f64) * 100.0); }

    eprintln!("\n  \x1b[1m{:<35} {:>6} {:>5} {:>4}\x1b[0m", "Species (CR)", "Enz", "PDB", "QC");
    eprintln!("  {}", "-".repeat(54));
    for r in results.iter().filter(|r| r.n_enzymes > 0) {
        let name = if r.scientific_name.len() > 33 { &r.scientific_name[..33] } else { &r.scientific_name };
        eprintln!("  {:<35} {:>6} {:>5} {:>4}", name, r.n_enzymes, r.n_pdb, r.n_qc);
    }

    // Save JSON
    let json = serde_json::json!({
        "terra_version": "0.5.0",
        "elapsed_seconds": elapsed.as_secs_f64(),
        "threads": n_threads,
        "species_scanned": n, "with_enzymes": s_with_enz,
        "total_enzymes": t_enz, "with_pdb": t_pdb,
        "quantum_critical": t_qc, "species_with_qc": s_qc,
        "qc_rate_pct": if t_pdb > 0 { (t_qc as f64 / t_pdb as f64) * 100.0 } else { 0.0 },
        "results": results.iter().map(|r| serde_json::json!({
            "species": r.scientific_name, "status": r.status,
            "enzymes": r.n_enzymes, "pdb": r.n_pdb, "qc": r.n_qc
        })).collect::<Vec<_>>(),
    });
    let _ = fs::write("terra-scan-results.json", serde_json::to_string_pretty(&json).unwrap_or_default());
    eprintln!("\n  Saved: terra-scan-results.json");
    eprintln!("\n  \x1b[36m\x1b[1mThe number that didn't exist.\x1b[0m\n");
}
