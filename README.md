# airtable-schema-generator

An NPM package designed to support React + Node apps that use Airtable as a backend!

- [What it does](#what-it-does)
- [Installation](#installation)
- [Running the script](#running-the-script)
- [CRUD Functions](#crud-functions)
- [Important Assumptions](#important-assumptions)
- [Manual Mode](#manual-mode)
- [Schema Metadata](#schema-metadata)
  * [Lookup Fields](#lookup-fields)
- [Record Transformations](#record-transformations)
- [Custom Filters and Sorts](#custom-filters-and-sorts)
- [Notes](#notes)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>


## What it does

After you install and setup the package, running `yarn generate-schema` (or `npm run generate-schema`), the script will scrape the Airtable website, download the latest schema for your app, and generate helper files. 

The helper files establish abstraction barriers for the rest of your project to follow. At the lowest level is **`airtable.js`**, which contains the functions that directly interact with airtable. Next, comes **`request.js`**, which are CRUD functions generated for _every_ table in your Airtable base. Finally is **`schema.js`**, a source of truth for your airtable schema.

Besides organizing your code's interactions with airtable via helper functions, the package also makes Airtable records easier to work with! The file `schema.js` contains mappings from Airtable Column Names to a more javascript-y version of them. For example, "First Name" becomes "firstName". It's aware of linked records as well; "Project Group" (which might be a one-to-one linked record column) will become "projectGroupId". 

In your codebase, you can use the javascript-y names, and the provided `airtable.js` file will automatically transform them to and from the actual Airtable column names! See more [here](#record-transformations)

Take a look at `schema.js` after you generate the files to get a sense of what the record transformations looks like for your Airtable base. Also explore `request.js` to see what functions you will be interacting with.

## Installation

#### 1) Add package as a dev dependency

`npm install --save-dev airtable-schema-generator`
or
`yarn add -D airtable-schema-generator`

#### 2) Choose a mode to use the package in

Options: 
1. **Auto**: You will provide your username and password in a `.airtable.env` file and the schema generator will scrape the airtable API page via an automated web browser
2. **Auto (Headless)**: Same as above, just without a visible web browser
3. **Manual**: You manually scrape airtable API page and save to file. No username and password needed

#### 3) Add generator settings to your package.json file

In your `package.json` add the following: 
```
"airtable-schema-generator": { 
  "mode": "<Desired Mode>",
  "output": "path/to/output/folder"
}
```

The `output` folder is where you'd like your generated files to live. 
The mode parameter accepts "auto", "auto-headless", and "manual"

**Other Parameters**: 
`input`: Only required if mode is set to Manual. Similar to `output`, specifies where `schemaRaw.json` can be found. Details below
`defaultView`: You can specify a default view to use for Read requests. Default is "Grid View"
`schemaMeta`: Where your metadata for schema generation lives. Details below
`envFileName`: The name of your environment variable file. Default is ".env"

#### 4) Create a `.env` file.

Create a file called `.env` in the root of your project and fill it in with the following: 

```
AIRTABLE_BASE_ID=
AIRTABLE_API_KEY=

## Needed if using Auto mode
AIRTABLE_EMAIL=
AIRTABLE_PASSWORD=
```


If you already have a `.env`, you can add these keys to the existing `.env` file. The Email and Password is only needed if using auto mode. This information is only saved on your computer in this hidden file.

If you'd like to use a different name for your environment file, pass it in in `package.json` under `envFileName`

#### 5) Add new env file to gitignore

Add the line `.env` to your `.gitignore` file

#### 6) Add convenient run script

Update your scripts object in `package.json` to include the following

```
"scripts": { 
  "generate-schema": "generate-airtable-schema"
}
```

Optional, add ` && pretty-quick` or your own prettifier command to the script to post-process the schema code. 

## Running the script

Run `npm run generate-schema` to re-generate your utility functions every time your airtable base updates!

## CRUD Functions

The functions it generates are below. The names of the functions are based on the table names. Names are naively pluralized by seeing if the last letter is an "s" or not.

`getAllRecords`: Gets all records of table

`getRecordById`: Gets a record given an ID

`getRecordsByIds`: Gets an array of records given an array of IDs

`getRecordsByAttribute`: Gets an array of records for which a certain attribute has a certain value

`createRecord`: Creates record given object containing fields to create

`updateRecord`: Updates record given an ID and object containing fields to update

`deleteRecord`: Deletes record given an ID

## Important Assumptions

This package makes assumptions of how you structure your Airtable. We note them here as best as possible
1. Linked Record relationship is accurate (one-to-many vs one-to-one)
2. Column Names do not have any special characters not accounted for (detailed under Record Transformations)
3. There are no two column names that would map to the same Javascript-y name. For example, "First Name" and "First Name?" would both map to "firstName" and isn't supported

## Manual Mode

If you'd prefer not to use one of the two automatic modes, the schema generator will instead look for a file called `schemaRaw.json` in your specified `input` folder. In order to generate your schema, do the following: 
1. Navigate to https://airtable.com/<YOUR_BASE_ID>/api/docs
2. Open the Javascript console (Cmd-Option-J on Chrome for Macs)
3. Run this command (it will copy the result to your clipboard): `copy(_.mapValues(application.tablesById, table => _.set(_.omit(table, ['sampleRows']),'columns',_.map(table.columns, item =>_.set(item, 'foreignTable', _.get(item, 'foreignTable.id'))))));`
4. Paste the result into your `schemaRaw.json` file
5. Run `yarn generate-schema`

You will need to repeat the steps above each time the airtable schema updates and you want to regenerate changes. If you do not update `schemaRaw.json`, the schema generation will not fail, but rather generate based on an outdated schema. 

Note: It's recommended to add `schemaRaw.json` to your `.gitignore` file

## Schema Metadata

Optionally, you can add a `schemaMeta` parameter to your `airtable-schema-generator` entry in `package.json`. This object lets you specify info about custom helper functions in `request.js`.  Sample Structure: 

```
{
  "User Login": {
    "lookupFields": ["Email"]
  },
  "Announcement": {
    "lookupFields": ["Project Group"]
  }
}

```

### Lookup Fields
The `lookupFields` meta attribute specifies which fields you would like to create a custom `getRecordsByAttribute` helper function for. For example, one of the functions the above `schemaMeta` would create might look like:
```
export const getAnnouncementsByProjectGroup = async value => {
  return getRecordsByAttribute(
    Tables.Announcement,
    Columns.Announcement.projectGroup,
    value
  );
};
```
This is in addition to the two default "get" functions created. 

## Record Transformations

To make working with records easier, this package includes functions that transform Airtable records after a Read action and before a Create/Update action. That transformation consists of the following: 

#### 1. Javascript-y Column Names

The transformation changes the column names of the record from the column name on Airtable (usually human readable) to the most likely variable name given basic Javascript conventions. 

Examples: 
- "First Name" -> "firstName"
- "Is Public?" -> "isPublic"

The current definition of the map from Human Readable to Javascript-y names is: 
1. Remove Special Characters: `(`, `)`, ` `, `"`, `'`, `?` (more can be added through a PR)
2. Lowercase Entire String
3. Split on spaces
4. Capitalize first character of each word except the first
5. Combine

#### 2. Accurate Linked Record Column Names

Linked Records on Airtable are usually named something like "Author" or "Project", which would make the corresponding javascript-y name "author" or "project". The most expressive name, however, given how they come back in the API response, would be "authorId" or "projectId". 

Record transformation accounts this, pluralizing it when the record is marked as a one-to-many relationship

#### 3. Wraps/Unwraps one-to-one Linked Record Values

Even though Airtable allows you to change a linked record to a one-to-many relationship, the values from the api are still considered an array, meaning you have to unwrap a value from an array when reading from airtable and re-wrap it when writing. 

Record transformation does this wrapping and unwrapping for you based on the type of relationship found on Airtable.

## Custom Filters and Sorts

For all `getAllRecords` functions, you can provide optional `filterByFormula` and `sort` props that will be passed to the base airtable functions. 

For any `getAllRecordsByAttribute` functions (generated by lookup fields), you can provide an optional `sort` prop.

## Notes

This generator was made originally to support Blueprint projects. Learn more about what we do here: https://calblueprint.org/
