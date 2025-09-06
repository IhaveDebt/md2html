use anyhow::{bail, Context, Result};
use clap::Parser;
use pulldown_cmark::{html, Options, Parser as MdParser};
use std::{fs, io::{self, Read, Write}, path::PathBuf};

#[derive(Parser, Debug)]
#[command(name="md2html", about="Convert Markdown to HTML")]
struct Args {
    /// Input markdown file (omit to read from STDIN)
    input: Option<PathBuf>,
    /// Output HTML file (omit to write to STDOUT)
    #[arg(short, long)]
    output: Option<PathBuf>,
    /// Add minimal HTML boilerplate
    #[arg(long)]
    page: bool,
}

fn main() -> Result<()> {
    let args = Args::parse();

    let mut md = String::new();
    if let Some(path) = args.input {
        md = fs::read_to_string(&path)
            .with_context(|| format!("Failed reading {:?}", path))?;
    } else {
        io::stdin().read_to_string(&mut md)?;
    }

    let mut opts = Options::empty();
    opts.insert(Options::ENABLE_TABLES);
    opts.insert(Options::ENABLE_TASKLISTS);

    let parser = MdParser::new_ext(&md, opts);
    let mut html_out = String::new();
    html::push_html(&mut html_out, parser);

    if args.page {
        html_out = format!(
r#"<!doctype html>
<html lang="en">
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>md2html</title>
<style>
body{{max-width: 800px; margin: 2rem auto; font-family: system-ui, Arial, sans-serif; line-height:1.6}}
code, pre{{font-family: ui-monospace, SFMono-Regular, Menlo, Consolas, "Liberation Mono", monospace}}
pre{{background:#f5f5f7; padding:1rem; border-radius:8px; overflow:auto}}
table{{border-collapse: collapse}} th,td{{border:1px solid #ddd; padding:.5rem}}
</style>
<body>
{}
</body>
</html>"#, html_out);
    }

    match args.output {
        Some(p) => fs::write(&p, html_out).with_context(|| format!("Failed writing {:?}", p))?,
        None => {
            let mut stdout = io::stdout().lock();
            stdout.write_all(html_out.as_bytes())?;
        }
    }

    Ok(())
}
