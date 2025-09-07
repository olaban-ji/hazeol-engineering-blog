+++
date = '2024-09-14'
draft = false
title = 'PostgreSQL and the 63-character limit'
+++

I recently had to set up a composite index on a model that uses Sequelize ORM to interface with PostgreSQL. Initially, the model looked like this;

<!--more-->

```js
const { Sequelize, DataTypes } = require('sequelize');

const sequelize = new Sequelize('database_name', 'username', 'password', {
  host: 'localhost',
  dialect: 'postgres',
});

const UserAccount = sequelize.define(
  'user_account',
  {
    columnA: {
      type: DataTypes.STRING,
      allowNull: true,
    },
    columnB: {
      type: DataTypes.STRING,
      allowNull: true,
    },
    columnC: {
      type: DataTypes.STRING,
      allowNull: true,
    },
    columnD: {
      type: DataTypes.STRING,
      allowNull: true,
    },
    columnE: {
      type: DataTypes.STRING,
      allowNull: true,
    },
    columnF: {
      type: DataTypes.STRING,
      allowNull: true,
    },
  },
  {
    /**
     * @note Composite Index
     */
    indexes: [
      {
        unique: true,
        fields: [
          'columnA',
          'columnB',
          'columnC',
          'columnD',
          'columnE',
          'columnF',
        ],
      },
    ],
  }
);

sequelize
  .sync()
  .then(() => {
    console.log(
      'UserAccount model has been synced with the PostgreSQL database.'
    );
  })
  .catch((err) => {
    console.error('Unable to sync the database:', err);
  });
```

Looks like this works fine right?! Well, partially.

In non-production environments, we use sequlize’s [synchronizing feature](https://sequelize.org/docs/v7/models/model-synchronization/) to automatically sync the database table with the model defined in the code. In this scenario, there’s a silent problem.

When sequelize synchronizes the model with the table for the first time, it works just fine. On subsequent restart of the application service, we’re going to run into an error like this;

```
https://sequelize.org/docs/v7/models/model-synchronization/
```

If you’re observant, you’ll notice that the relation that is being claimed to exist is somewhat cut out. Where did column_f go all of a sudden and why do we have colu in its place?

Turns out, PostgreSQL has a [63-character limit](https://www.postgresql.org/docs/current/sql-syntax-lexical.html#SQL-SYNTAX-IDENTIFIERS:~:text=The%20system%20uses%20no%20more%20than%20NAMEDATALEN%2D1) that is imposed by [NAMEDATALEN](https://pgpedia.info/n/NAMEDATALEN.html) which is a constant that defines the maximum length of database identifiers.

This is a breakdown of what happens that leads to this error;

* During the first application start, PostgreSQL auto-generates an index name since one wasn’t provided which would be user_accounts_column_a_column_b_column_c_column_d_column_e_column_f and truncates it to user_accounts_column_a_column_b_column_c_column_d_column_e_colu considering that the auto-generated identifier name (67 character count) exceeds the character limit.
* When the app restarts, sequelize will try to create the index again using the auto-generated index name, truncate it to 63 characters and that’s when we’d get the above error because the truncated version of this index name already exists.
To mitigate this, add a custom index name property that complies with the character limit to the composite index in the model definition. Like this;

```js
const { Sequelize, DataTypes } = require('sequelize');

const sequelize = new Sequelize('database_name', 'username', 'password', {
  host: 'localhost',
  dialect: 'postgres',
});

const UserAccount = sequelize.define(
  'user_account',
  {
    columnA: {
      type: DataTypes.STRING,
      allowNull: true,
    },
    columnB: {
      type: DataTypes.STRING,
      allowNull: true,
    },
    columnC: {
      type: DataTypes.STRING,
      allowNull: true,
    },
    columnD: {
      type: DataTypes.STRING,
      allowNull: true,
    },
    columnE: {
      type: DataTypes.STRING,
      allowNull: true,
    },
    columnF: {
      type: DataTypes.STRING,
      allowNull: true,
    },
  },
  {
    indexes: [
      {
        /**
         * @note Custom Index Name
         */
        name: 'user_accounts_columnA_columnB_columnC_columnD_columnE_columnF',
        unique: true,
        fields: [
          'columnA',
          'columnB',
          'columnC',
          'columnD',
          'columnE',
          'columnF',
        ],
      },
    ],
  }
);

sequelize
  .sync()
  .then(() => {
    console.log(
      'UserAccount model has been synced with the PostgreSQL database.'
    );
  })
  .catch((err) => {
    console.error('Unable to sync the database:', err);
  });
```

user_accounts_columnA_columnB_columnC_columnD_columnE_columnF has a character count of 61 which is just under 63 characters so we wouldn’t have a problem with database conflict on subsequent application restarts.