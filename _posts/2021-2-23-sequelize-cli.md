---
layout: post
title: Using sequelize-cli to version control database state changes
---

If you don’t know what’s `npx`, please check out this [youtube video](https://www.youtube.com/watch?v=55WaAoZV_tQ).

**Problem Statement**: 

As a developer, when there is a need to change the SQL database, be it its schema, or stored procedure, or db events, we are very tempted to change it directly in the database. However, it’s not a very good practice, because when we try to deploy the solution from local environment to dev environment, or from dev environment to production environment, we may easily forget what we did in the source environment. Manually keeping track of all the changes in different environments is simply time-consuming, unreliable and prone to errors.

![Messy room](/images/sequelize-files.png)

So a version control on database changes are very necessary and can help us (without memorising all the changes done in the source environments) reproduce the changes again in the target environment. 

**Solution**:

sequelize-cli and migrations files to rescue.

Assume that you are in an npm project repository, let’s install sequelize-cli first in the local repository.

```
npm install --save-dev sequelize-cli
```

Let’s create a migration skeleton first.

```
npx sequelize migration:create --name test-npx
```

It will create a migration skeleton under migrations folder in your project with the prefix of the current date and timestamp. So that the files under migrations folder can be sorted using the filename by default in the chronological order. 

For example:

Once the migration skeleton is created, you can start editing the skeleton to put down what you want to migrate.

Editing the actual file: 

```
'use strict';

module.exports = {
  up: async (queryInterface, Sequelize) => {
    /**
     * Add altering commands here.
     *
     * Example:
     * await queryInterface.createTable('users', { id: Sequelize.INTEGER });
     */
  },

  down: async (queryInterface, Sequelize) => {
    /**
     * Add reverting commands here.
     *
     * Example:
     * await queryInterface.dropTable('users');
     */
  }
};
```

sequelize-cli will generate the skeleton of the migration file, which contains both up (commit) and down (rollback) functions. Both of them should return a promise, so using async / await is preferred. 

Let’s see some examples starting from creating table in SQL Database.

Caveat: A model in Sequelize has a name. This name does not have to be the same name of the table it represents in the database. Usually, models have singular names (such as User) while tables have pluralized names (such as Users), although this is fully configurable.

DB table creation:
```
'use strict';
module.exports = {
    up: (queryInterface, Sequelize) => {
        return queryInterface.createTable('prizes', {
            id: {
                type: Sequelize.INTEGER,
                primaryKey: true,
                autoIncrement: true,
                allowNull: false
            }, 
            image_url: {
                type: Sequelize.STRING,
                allowNull: true
            },
            quantity_limit: {
                type: Sequelize.INTEGER.UNSIGNED,
                allowNull: false
            },
            description: { 
                type: Sequelize.STRING,
                allowNull: true
            },
            terms_conditions: { 
                type: Sequelize.TEXT,
                allowNull: true
            },
            how_to_redeem: {
                type: Sequelize.TEXT,
                allowNull: true
            },
            total_inventory: {
                type: Sequelize.INTEGER.UNSIGNED,
                allowNull: false
            },
            name: { 
                type: Sequelize.STRING,
                allowNull: false
            },
            type: { 
                type: Sequelize.STRING,
                allowNull: false
            },
            category: { 
                type: Sequelize.STRING,
                allowNull: false
            },
            point1: {
                type: Sequelize.INTEGER.UNSIGNED,
                allowNull: false
            },
            point2: {
                type: Sequelize.INTEGER.UNSIGNED,
                allowNull: false
            },
            point3: {
                type: Sequelize.INTEGER.UNSIGNED,
                allowNull: false
            },
            type: { 
                type: Sequelize.STRING,
                allowNull: false
            },
            category: { 
                type: Sequelize.STRING,
                allowNull: false
            },
            publication: { 
                type: Sequelize.STRING,
                allowNull: false
            },
            createdAt: {
                allowNull: false,
                type: Sequelize.DATE
            },
            updatedAt: {
                allowNull: false,
                type: Sequelize.DATE
            }
        });
    },
    down: (queryInterface, Sequelize) => {
        return queryInterface.dropTable('prizes');
    }
};
```
Add a new DB table field:
```
'use strict';

module.exports = {
    up: (queryInterface, Sequelize) => {
        return queryInterface.addColumn(
            'prizes', // table name
            'center_image_url', // new field name
            {
                type: Sequelize.STRING,
                allowNull: true,
            }
        );
    },

    down: (queryInterface, Sequelize) => {
        return queryInterface.removeColumn('prizes', 'center_image_url');
    }
};
```
Change the existing DB table field type:
```
'use strict';

module.exports = {
    up: (queryInterface, Sequelize) => {
        return queryInterface.changeColumn('prizes', 'description', {
            type: Sequelize.TEXT,
        });
    },

    down: (queryInterface, Sequelize) => {
        return queryInterface.changeColumn('prizes', 'description', {
            type: Sequelize.STRING,
        });
    }
};
```
Add a table index:
```
'use strict';

module.exports = {
    up: async (queryInterface, Sequelize) => {
        await queryInterface.addIndex(
            'prizes',
            ['publication'],
            {
                indexType: 'BTREE'               
            }
        );
    },

    down: async (queryInterface, Sequelize) => {
        await queryInterface.removeIndex('prizes', ['publication']);
    }
};
```
Currently the indexName option doesn’t really work based on my experiment, so it will use the default naming convention, TABLENAME_COLUMNNAME, to name the index.

Add a new stored procedure:
```
'use strict';

module.exports = {
    up: async (queryInterface, Sequelize) => {
        await queryInterface.sequelize.query('DROP PROCEDURE IF EXISTS update_prizes_active_inventory;');
        
        return queryInterface.sequelize.query(`
        CREATE PROCEDURE update_prizes_active_inventory()
        BEGIN
          DECLARE _rollback BOOL DEFAULT 0;
          DECLARE CONTINUE HANDLER FOR SQLEXCEPTION SET _rollback = 1;
          SET SQL_SAFE_UPDATES = 0;
          START TRANSACTION;
            UPDATE prizes SET active_inventory = (
              SELECT COUNT(prizeId)
              FROM redemption_codes
              WHERE redemption_codes.prizeId = prizes.id AND
                (transactionId IS NULL) AND
                activatedAt <= UTC_TIMESTAMP()
              AND UTC_TIMESTAMP() <= expiryAt
            ) WHERE type = 'E' and status = 'Active';
          IF _rollback THEN
            ROLLBACK;
          ELSE
          COMMIT;
          END IF;
          SET SQL_SAFE_UPDATES = 1;
        END;`);
    },

    down: (queryInterface, Sequelize) => {
        return queryInterface.sequelize.query('DROP PROCEDURE IF EXISTS update_prizes_active_inventory;');
    }
};
```
Add a db scheduled event:

Caveat: If you create events from browser or IDE, it has the session time zone set to your local time zone, so the event is scheduled based on your local timezone. While if you create the event from the migration file, there is no such session info (unless you specify it like in the line No.5 in the following code snippet) so it uses UTC timezone by default. reference
```
'use strict';

module.exports = {
    up: async (queryInterface, Sequelize) => {
        await queryInterface.sequelize.query('SET @@session.time_zone="+08:00";');
        await queryInterface.sequelize.query('DROP EVENT IF EXISTS event_update_prizes_active_inventory;');
        return queryInterface.sequelize.query(`
        CREATE EVENT event_update_prizes_active_inventory
        ON SCHEDULE EVERY 1 DAY STARTS (DATE(CONVERT_TZ(UTC_TIMESTAMP(), '+00:00', '+08:00')) + INTERVAL 1 DAY)
        COMMENT 'UPDATE Active Inventory'
        DO CALL update_prizes_active_inventory();
        `);
    },

    down: (queryInterface, Sequelize) => {
        return queryInterface.sequelize.query('DROP EVENT IF EXISTS event_update_prizes_active_inventory;');
    }
};
```
The final step after you add the migrations files is to run the migration.

Sequelize provides a very elegant way to keep track of what migrations files have already run in the db and what are not in the table called SequelizeMeta. Whenever a migration is run, it will check against this table to see which files haven’t migrated yet and run these migration files to the database you specified in the url flag.

Running the following command will migrate all the files into local MySQL DB instance.
```
npx sequelize db:migrate --url 'mysql://root:root@localhost:8889/rewards'
```
*Voila!*

All the changes should be migrated into the local MySQL. Let’s party!

P.S. You can also create some seed data into database by default. It can be realised by sequelize-cli too. The complete tutorial on how to generate seed data is in the [link](https://sequelize.org/master/manual/migrations.html#creating-the-first-seed).
