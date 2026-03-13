# Controllers

Mapping controller functions to GraphQL fields is done using GraphQL annotations. These annotations can map any function to any parent object making them a powerful way to build out your API.

## Creating a Query or Mutation Field

To map a Query or Mutation field start define a GraphQL controller class in `src/Controller/GraphQL`, we will call ours `GraphQLController.php` but as your API becomes more complex you will almost certainly want to separate these into entity-specific controllers for clarity.

```php
<?php

namespace Application\Controller\GraphQL;

class GraphQLController {
}
```

Now that we have our controller class we can start adding fields to our graphql API using annotations, let's add a function to get all posts from our hypothetical `Post` entity.

```php
<?php

namespace Application\Controller\GraphQL;

use Doctrine\ORM\EntityManagerInterface;
use ForestCityLabs\Framework\GraphQL\Annotation as GraphQL;

class GraphQLController {
  #[GraphQL\Owner('Query')]
  #[GraphQL\Field(type: 'Post')]
  public function getPosts(
    EntityManagerInterface $em
  ): array {
    return $em->getRepository(Post::class)->findAll();
  }
}
```


:::note

Controller functions use dependency injection to fill in missing parameters, so the `EntityManagerInterface` is automatically injected into the function for you!

:::

This will create a field on the Query object for `getPosts` that returns an array of posts, the schema would look as follows:

```graphql
type Query {
  getPosts: [Post!]!
}
```
