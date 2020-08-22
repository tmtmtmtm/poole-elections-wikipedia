
The code and queries etc here are unlikely to be updated as my process
evolves. Later repos will likely have progressively different approaches
and more elaborate tooling, as my habit is to try to improve at least
one part of the process each time around.

---------

Step 1: Configure config.json
=============================

All the relevant metadata now lives in config.json: ideally nothing will
need tweaked after this. We need to be careful here to get the history
of Wikidata IDs for the constituency correct.

Step 1: Scrape the results
==========================

```sh
bundle exec ruby scraper.rb config.json | tee wikipedia.csv
```

Step 2: Check for missing party IDs
===================================

```sh
xsv search -v -s party 'Q' wikipedia.csv
```

One party linked to existing Wikidata item, and one new party created:

* wb ce create-new-party.js "Servicemen & Citizen Association"

Ste 3: Check for missing election IDs
=====================================

```sh
xsv search -v -s election 'Q' wikipedia.csv | xsv select electionLabel | uniq
```

Three new by-election items created:

* wb ce create-by-election.js "1884 Poole by-election" 1884-04-19
* wb ce create-by-election.js "1850 Poole by-election" 1850-09-24
* wb ce create-by-election.js "1835 Poole by-election" 1835-05-21

Step 4: Generate possible missing person IDs
============================================

```sh
xsv search -v -s id 'Q' wikipedia.csv | xsv select name | tail +2 |
  sed -e 's/^/"/' -e 's/$/"@en/' | paste -s - |
  xargs -0 wd sparql find-candidates.js |
  jq -r '.[] | [.name, .item.value, .election.label, .constituency.label, .party.label] | @csv' |
  tee candidates.csv
```

Step 5: Combine Those
=====================

```sh
xsv join -n --left 2 wikipedia.csv 1 candidates.csv | xsv select '10,1-8' | sed $'1i\\\nfoundid' | tee combo.csv
```

Step 6: Generate QuickStatements commands
=========================================

```sh
bundle exec ruby generate-qs.rb config.json | tee commands.qs
```

Then sent to QuickStatements as https://editgroups.toolforge.org/b/QSv2T/1598128423484/
