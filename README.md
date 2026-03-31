# @codenameryuu/adonis-lucid-soft-deletes

This addon adds the functionality to soft deletes Lucid Models through the `deleted_at` flag in AdonisJS.

## Requirement

* Adonis Js 7
* Lucid 22 or higher

## Installation

* Install the package

```bash
yarn add @codenameryuu/adonis-lucid-soft-deletes
```

* Configure the package

```bash
node ace configure @codenameryuu/adonis-lucid-soft-deletes
```

* Make sure to register the provider inside `adonisrc.ts` file.

```typescript
providers: [
  // ...
  () => import('@codenameryuu/adonis-lucid-soft-deletes/provider'),
],
```

You should add the `deleted_at` column to your database tables for models with soft deletes.

```typescript
// migrations/1234566666_users.ts
import { BaseSchema } from '@adonisjs/lucid/schema'

export default class extends BaseSchema {
  protected tableName = 'products'

  async up() {
    this.schema.createTable(this.tableName, (table) => {
      // ...
      table.timestamp('deleted_at').nullable()
    })
  }
  // ...
}
```

### Applying Soft Deletes to a Model

```typescript
import { compose } from '@adonisjs/core/helpers'
import { SoftDeletes } from '@codenameryuu/adonis-lucid-soft-deletes'

export default class Product extends compose(BaseModel, SoftDeletes) {
  // ...columns and props
}
```

Now, when you call the `.delete()` method on the model, the `deleted_at` ( `customDeletedAtColumn` ) column
will be set to the current date and time. However, the model's database record will be left in the table.

```typescript
import type { HttpContext } from '@adonisjs/core/http'
import Product from '#models/product'

export default class ProductsController {
  /**
   * Delete product by id
   * DELETE /products/:id
   */
  async destroy({ request, response }: HttpContext) {
    const product = await Product.findOrFail(request.input('id'))
    await product.delete()

    return product // or response.noContent()
  }
}
```

> :boom: Soft delete only works for model instances. `await Product.query().delete()` as before
> will delete models from database

:point_right: When querying a model that uses soft deletes, the soft deleted models
will automatically be excluded from all query results.

To determine if a given model instance has been soft deleted, you may use the `.trashed` getter:

```typescript
import type { HttpContext } from '@adonisjs/core/http'
import Product from '#models/product'

export default class ProductsController {
  /**
   * Get product by id
   * GET /products/:id
   */
  async show({ request, response }: HttpContext) {
    const product = await Product.withTrashed().where('id', request.input('id')).firstOrFail()
    if (product.trashed) {
      return response.forbidden()
    }
    return product
  }
}
```

### Set custom column name for `deletedAt`

```typescript
import { compose } from '@adonisjs/core/helpers'
import { SoftDeletes } from '@codenameryuu/adonis-lucid-soft-deletes'

export default class Product extends compose(BaseModel, SoftDeletes) {
  // ...columns and props

  @column.dateTime({ columnName: 'customDeletedAtColumn' })
  declare deletedAt: DateTime | null
}
```

### Restoring Soft Deleted Models

To restore a soft deleted model, you may call the `.restore()` method on a model instance.
Also, method `.restore()` exists after methods `.withTrashed()` and `.onlyTrashed()`

The `restore` method will set the model's `deleted_at` column to `null` :

```ts
import type { HttpContext } from '@adonisjs/core/http'
import Product from '#models/product'

export default class TrashProductsController {
  /**
   * Update trashed product by id
   * PUT /trash/products/:id
   */
  async update({ request }: HttpContext) {
    const product = await Product.withTrashed().where('id', request.input('id')).firstOrFail()
    await product.restore()

    return product

    // or

    await Product.withTrashed().where('id', request.input('id')).restore()
    await Product.query().withTrashed().where('id', request.input('id')).restore()
  }
}
```

### Permanently Deleting Models

Sometimes you may need to truly remove a model from your database.
You may use the `.forceDelete()` method to permanently remove a soft deleted model from the database table:

```typescript
import type { HttpContext } from '@adonisjs/core/http'
import Product from '#models/product'

export default class ProductsController {
  /**
   * Delete product by id
   * DELETE /products/:id
   */
  async destroy({ request, response }: HttpContext) {
    const product = await Product.findOrFail(request.input('id'))
    await product.forceDelete()

    return response.noContent()
  }
}
```

### Including Soft Deleted Models

As noted above, soft deleted models will automatically be excluded from query results.
However, you may force soft deleted models to be included in a query's results
by calling the `.withTrashed()` method on the model:

```typescript
import type { HttpContext } from '@adonisjs/core/http'
import Product from '#models/product'

export default class ProductsController {
  /**
   * Get a list products
   * GET /products?withTrashed=1
   */
  async index({ request }: HttpContext) {
    return Product.query()
      .if(request.input('withTrashed'), (query) => {
        query.withTrashed()
      })
      .paginate(request.input('page', 1), request.input('perPage', 10))
  }
}
```

### Retrieving only Soft Deleted Models

The `.onlyTrashed()` method will retrieve **only** soft deleted models:

```typescript
import type { HttpContext } from '@adonisjs/core/http'
import Product from '#models/product'

export default class TrashProductsController {
  /**
   * Get a list trashed products
   * GET /trash/products
   */
  async index({ request }: HttpContext) {
    return Product.onlyTrashed().paginate(request.input('page', 1), request.input('perPage', 10))
  }
}
```

### Soft Deletes methods

Methods `.withTrashed()` , `.onlyTrashed()` and `.restore()` also available
in ModelQueryBuilder for models with soft delete, example:

```typescript
await Product.query().withTrashed().paginate(request.input('page', 1), request.input('perPage', 10))
await Product.query().onlyTrashed().paginate(request.input('page', 1), request.input('perPage', 10))
```

## License

This package is open-sourced software licensed under the [MIT license](LICENSE.md).
