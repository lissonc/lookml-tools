# LookML Linter

As part of this tool suite, we include a LookML linter--a linter is a tool that checks that code conforms to some specified rules, such as formatting or naming conventions. While there are at least two other open source linters [https://github.com/WarbyParker/lookmlint]() and [https://looker-open-source.github.io/look-at-me-sideways/rules.html](), they don't include our particular list of coding standards so either we would have to code our checks into their framework or code our own. As it only amounts to 200 lines of code, and we can reuse code from the LookML updater, we went with the latter option.

## How it works
The `config/linter/config_linter.json` specifies which set of files to apply the linter to, by specifying a file glob. For instance, 

```
    "infile_globs": [
        "/Users/carl.anderson/code/core-analytics-looker/*.view.lkml"
    ],
```
specifies it to run on all view files in Carl's `core-analytics-looker` folder.

The config file also specifies a set of rules to be run. For instance, this

```
"rules": {
        "file_level_rules" : [
            {"name": "DataSourceRule", "run": true},
            {"name": "OneViewPerFileRule", "run": true},
            {"name": "FilenameViewnameMatchRule", "run": true}
        ],
        "field_level_rules": [
            {"name": "DescriptionRule", "run": true},
            {"name": "DrillDownRule", "run": true},
            {"name": "YesNoNameRule", "run": true},
            {"name": "CountNameRule", "run": true},
            {"name": "AllCapsRule", "run": true}
            {"name": "LexiconRule", "run": true}
        ]
    },

```
specifies to run eight different rules. The rules are in two groups:

 - file-level rules: a rule that is applied to the file once, e.g. does the file contain just one view
 - field-level rules: a rule that is applied to each and every `dimension`, `dimension_group`, and `measure`. For instance, is the name of each field in ALL CAPS? If a rule is not relevant, e.g. a measure only rule is applied to a dimension, the rule is skipped and no output is added to the output files (see below).

The tool makes use of Fabio's LookML parser (https://github.com/fabio-looker/node-lookml-parser)

```
brew install node   # if on mac
npm install -g lookml-parser
```
which converts LookML into a JSON object that can be interrogated easily with the set of rules.

The results are, optionally, written to two CSVs:

```
    "output": 
        "csv": {
            "file_output": "linter_file_report.csv",
            "field_output": "linter_field_report.csv"
        }
    }
```
The former consists of one result (one line of CSV) per file-level rule per input file. The latter consists of one result per applicable rule per `dimension`, `dimension_group`, or `measure`. 

They can also be written to BigQuery:

```
    "output": {
        "csv": {
            "file_output": "linter_file_report.csv",
            "field_output": "linter_field_report.csv"
        },
        "bigquery": {
            "target_bucket_name": "your_bucket",
            "bucket_folder": "your_folder",
            "project_id": "your-project",
            "dataset": "your-dataset",
            "file_destination_table": "lookml_linter_file_report2",
            "field_destination_table": "lookml_linter_field_report2"
        }
    }
```

## Installation

This code makes use of Fabio's nade-based LookML parser (https://github.com/fabio-looker/node-lookml-parser)

```
brew install node   # if on mac
npm install -g lookml-parser
```

You will need to set the path of the `lookml-parser` binary in the linter config file. For example:

```
{
    "parser": "/usr/local/bin/lookml-parser",
    ...
}
```

## Running

To run, 

```
python run_linter.py --config config/linter/config_linter.json
```
and this saves output files such as:

`linter_file_report.csv`:

```
time,file,rule,passed
2019-04-09 09:05:50,test_view_rewards.view.lkml,DataSourceRule,0
2019-04-09 09:05:50,test_view_rewards.view.lkml,OneViewPerFileRule,1
2019-04-09 09:05:50,test_view_rewards.view.lkml,FilenameViewnameMatchRule,1
2019-04-09 09:05:50,winsbyday_international.view.lkml,DataSourceRule,1
2019-04-09 09:05:50,winsbyday_international.view.lkml,OneViewPerFileRule,1
...
```

and 

`linter_field_report.csv`:

```
time,file,rule,type,fieldname,passed
2019-04-09 09:05:50,winsbyday_international.view.lkml,DescriptionRule,dimension,activity,0
2019-04-09 09:05:50,winsbyday_international.view.lkml,DrillDownRule,dimension,activity,0
2019-04-09 09:05:50,winsbyday_international.view.lkml,AllCapsRule,dimension,activity,1
2019-04-09 09:05:50,winsbyday_international.view.lkml,DescriptionRule,dimension,country,0
2019-04-09 09:05:50,winsbyday_international.view.lkml,DrillDownRule,dimension,country,0
2019-04-09 09:05:50,winsbyday_international.view.lkml,AllCapsRule,dimension,country,1
```

## The Rules

File-level rules:

 - **DataSourceRule**: does source contain a sql_table_name or derived_table?
 - **OneViewPerFileRule**: is there is one view only in this file?
 - **FilenameViewnameMatchRule**: does the name of the file match the name of the view?

Field-level rules:

 - **DescriptionRule**: does the dimension, dimension_group, or measure have a description?
 - **DrillDownRule**: does measure have drill downs? 
 - **YesNoNameRule**: if this is a yesno dimension, does name start with 'is_'?
 - **CountNameRule**: if this is a measure of type count, does name end with '_count'?
 - **AllCapsRule**: is the name not ALL CAPS?
 - **LexiconRule**: does the name or description (if any) of dimension, dimension_group, or measures mention "subscriber", "subscription", or "studio"? If so, that's a fail.

Other rules:
 - **NoOrphansRule**: each view should be referenced by an explore. If not, the file is an orphan. This is a special rule in that assessment of whether a file has passed can only be ascertained after all files in the repo have been processed.

To add a new rule, it needs to implement the very simple `lkmltools.linter.rule` interface (excepting any special rules).