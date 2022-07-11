<a href="https://nhogs.com"><img src="https://nhogs.com/nhogs_64.png" align="right" alt="nhogs-logo" title="NHOGS Interactive"></a>

# @nhogs/nestjs-neo4j

## Description

[Neo4j](https://neo4j.com/) module for [Nest.js](https://github.com/nestjs/nest).

[![e2e-test](https://github.com/nhogs/nestjs-neo4j/actions/workflows/e2e-test.yml/badge.svg)](https://github.com/Nhogs/nestjs-neo4j/actions/workflows/e2e-test.yml)
[![Docker-tested-version](https://img.shields.io/badge/E2e%20tests%20on-neo4j%3A4.4.8--enterprise-blue?logo=docker)](https://hub.docker.com/layers/neo4j/library/neo4j/4.4.8-enterprise/images/sha256-c6e315b42b260c81177c7ae5645e4fec5be7b0c9febada7d38df8cd698cd1a3b?context=explore)
[![Maintainability](https://api.codeclimate.com/v1/badges/2de17798cf9b4d9cfd83/maintainability)](https://codeclimate.com/github/Nhogs/nestjs-neo4j/maintainability)
[![Test Coverage](https://api.codeclimate.com/v1/badges/2de17798cf9b4d9cfd83/test_coverage)](https://codeclimate.com/github/Nhogs/nestjs-neo4j/test_coverage)

### Peer Dependencies

[![npm peer dependency version NestJS)](https://img.shields.io/npm/dependency-version/@nhogs/nestjs-neo4j/peer/@nestjs/core?label=Nestjs&logo=nestjs&logoColor=e0234e)](https://github.com/nestjs/nest)
[![npm peer dependency version neo4j-driver)](https://img.shields.io/npm/dependency-version/@nhogs/nestjs-neo4j/peer/neo4j-driver?label=neo4j-driver&logo=neo4j)](https://github.com/neo4j/neo4j-javascript-driver)

## Installation
![npm](https://img.shields.io/npm/v/@nhogs/nestjs-neo4j?logo=npm)
```bash
$ npm i --save @nhogs/nestjs-neo4j
```

## Usage

### In static module definition:

```typescript
@Module({
  imports: [
    Neo4jModule.forRoot({
      scheme: 'neo4j',
      host: 'localhost',
      port: '7687',
      database: 'neo4j',
      username: 'neo4j',
      password: 'test',
      global: true, // to register in the global scope
    }),
    CatsModule,
  ],
})
export class AppModule {}
```

### In async module definition:

```typescript
@Module({
  imports: [
    Neo4jModule.forRootAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (configService: ConfigService): Neo4jConfig => ({
        scheme: configService.get('NEO4J_SCHEME'),
        host: configService.get('NEO4J_HOST'),
        port: configService.get('NEO4J_PORT'),
        username: configService.get('NEO4J_USERNAME'),
        password: configService.get('NEO4J_PASSWORD'),
        database: configService.get('NEO4J_DATABASE'),
      }),
      global: true,
    }),
    PersonModule,
    ConfigModule.forRoot({
      envFilePath: './test/src/.test.env',
    }),
  ],
})
export class AppAsyncModule {}
```

### [🔗 Neo4jService](/lib/service/neo4j.service.ts) :

```typescript
@Injectable()
/** See https://neo4j.com/docs/api/javascript-driver/current/ for details ...*/
export class Neo4jService implements OnApplicationShutdown {
  constructor(
    @Inject(NEO4J_CONFIG) private readonly config: Neo4jConfig,
    @Inject(NEO4J_DRIVER) private readonly driver: Driver,
  ) {}

  /** Verifies connectivity of this driver by trying to open a connection with the provided driver options...*/
  verifyConnectivity(options?: { database?: string }): Promise<ServerInfo> {...}

  /** Regular Session. ...*/
  getSession(options?: SessionOptions): Session {...}

  /** Reactive session. ...*/
  getRxSession(options?: SessionOptions): RxSession {...}

  /** Run Cypher query in regular session and close the session after getting results. ...*/
  run(
    query: Query,
    sessionOptions?: SessionOptions,
    transactionConfig?: TransactionConfig,
  ): Promise<QueryResult> {...}

  /** Run Cypher query in reactive session. ...*/
  rxRun(
    query: Query,
    sessionOptions?: SessionOptions,
    transactionConfig?: TransactionConfig,
  ): RxResult {...}

  /** Returns constraints as runnable Cypher queries defined with decorators on models. ...*/
  getCypherConstraints(label?: string): string[] {...}

  onApplicationShutdown() {
    return this.driver.close();
  }
}
```

```typescript
/**
 * Cat Service example 
 */

@Injectable()
export class CatService {
  constructor(private readonly neo4jService: Neo4jService) {}

  async create(cat: Cat): Promise<Cat> {
    const queryResult = await this.neo4jService.run(
      {
        cypher: 'CREATE (c:`Cat`) SET c=$props RETURN properties(c) AS cat',
        parameters: {
          props: cat,
        },
      },
      { write: true },
    );

    return queryResult.records[0].toObject().cat;
  }

  async findAll(): Promise<Cat[]> {
    return (
      await this.neo4jService.run({
        cypher: 'MATCH (c:`Cat`) RETURN properties(c) AS cat',
      })
    ).records.map((record) => record.toObject().cat);
  }
}
```

### Run with reactive session

```typescript
neo4jService
  .rxRun({ cypher: 'MATCH (n) RETURN count(n) AS count' })
  .records()
  .subscribe({
    next: (record) => {
      console.log(record.get('count'));
    },
    complete: () => {
      done();
    },
  });
```

### Define constraints with decorators on Dto
https://neo4j.com/docs/cypher-manual/current/constraints/

- @NodeKey():
  - Node key constraints
- @Unique():
  - Unique node property constraints
- @NotNull():
  - Node property existence constraints
  - Relationship property existence constraints

[🔗 Constraint decorators - source code](/lib/decorator/constraint.decorator.ts)

```typescript
@Node({ label: 'Person' })
export class PersonDto {
  @NodeKey({ additionalKeys: ['firstname'] })
  name: string;

  @NotNull()
  firstname: string;

  @NotNull()
  @Unique()
  surname: string;

  @NotNull()
  age: number;
}

@Relationship({ type: 'WORK_IN' })
export class WorkInDto {
  @NotNull()
  since: Date;
}
```

Will generate the following constraints:

```cypher
CREATE CONSTRAINT `person_name_key` IF NOT EXISTS FOR (p:`Person`) REQUIRE (p.`name`, p.`firstname`) IS NODE KEY
CREATE CONSTRAINT `person_firstname_exists` IF NOT EXISTS FOR (p:`Person`) REQUIRE p.`firstname` IS NOT NULL
CREATE CONSTRAINT `person_surname_unique` IF NOT EXISTS FOR (p:`Person`) REQUIRE p.`surname` IS UNIQUE
CREATE CONSTRAINT `person_surname_exists` IF NOT EXISTS FOR (p:`Person`) REQUIRE p.`surname` IS NOT NULL
CREATE CONSTRAINT `person_age_exists` IF NOT EXISTS FOR (p:`Person`) REQUIRE p.`age` IS NOT NULL

CREATE CONSTRAINT `work_in_since_exists` IF NOT EXISTS FOR ()-[p:`WORK_IN`]-() REQUIRE p.`since` IS NOT NULL
```

### Extends Neo4j Model Services helpers to get basic CRUD methods for node or relationships:

```mermaid
classDiagram
class Neo4jModelService~T~
<<abstract>> Neo4jModelService
class Neo4jNodeModelService~N~
<<abstract>> Neo4jNodeModelService
class Neo4jRelationshipModelService~R~
<<abstract>> Neo4jRelationshipModelService
Neo4jModelService : string label*
Neo4jModelService : runCypherConstraints()
Neo4jModelService <|--Neo4jNodeModelService
Neo4jNodeModelService : nodeCRUDFunctions()
Neo4jModelService <|--Neo4jRelationshipModelService
Neo4jRelationshipModelService : relationshipCRUDFunctions()
```

See source code for more details:

- [🔗 Neo4jModelService](lib/service/neo4j.model.service.ts)
- [🔗 Neo4jNodeModelService](lib/service/neo4j.node.model.service.ts)
- [🔗 Neo4jRelationshipModelService](lib/service/neo4j.relationship.model.service.ts)

```typescript
export abstract class Neo4jNodeModelService<T> extends Neo4jModelService<T>{
  async create(props: Partial<T>): Promise<T>{...}
  async merge(props: Partial<T>): Promise<T>{...}
  async update(match: Partial<T>, update: Partial<T>, mutate = true): Promise<T>{...}
  async delete(props: Partial<T>): Promise<T[]>{...}
  async findAll(params?: {
    skip?: number;limit?: number;orderBy?: string; descending?: boolean;
  }): Promise<T[]>{...}
  async findBy(params: {
    props: Partial<T>; skip?: number; limit?: number;  orderBy?: string; descending?: boolean;
  }): Promise<T[]>{...}
}

export abstract class Neo4jRelationshipModelService<R> extends Neo4jModelService<R> {
  async create<F, T>(
          props: Partial<R>,
          fromProps: Partial<F>,
          toProps: Partial<T>,
          fromService: Neo4jNodeModelService<F>,
          toService: Neo4jNodeModelService<T>,
          merge = false,
  ): Promise<R[]>{...}
}
```

#### Examples:

Look at [🔗 E2e tests usage](spec/e2e) for more details

```typescript
/**
 * Cat Service example
 */

@Injectable()
export class CatsService extends Neo4jNodeModelService<Cat> {
  constructor(protected readonly neo4jService: Neo4jService) {
    super();
  }

  label = 'Cat';
  logger = undefined;

  fromNeo4j(model: Record<string, any>): Cat {
    return super.fromNeo4j({
      ...model,
      age: model.age.toNumber(),
    });
  }

  toNeo4j(cat: Record<string, any>): Record<string, any> {
    let result: Record<string, any> = { ...cat };

    if (!isNaN(result.age)) {
      result.age = int(result.age);
    }

    return super.toNeo4j(result);
  }

  // Add a property named 'created' with timestamp on creation
  protected timestamp = 'created';

  async findByName(params: {
    name: string;
    skip?: number;
    limit?: number;
    orderBy?: string;
    descending?: boolean;
  }): Promise<Cat[]> {
    return super.findBy({
      props: { name: params.name },
      ...params,
    });
  }

  async searchByName(params: {
    search: string;
    skip?: number;
    limit?: number;
  }): Promise<[Cat, number][]> {
    return super.searchBy({
      prop: 'name',
      terms: params.search.split(' '),
      skip: params.skip,
      limit: params.limit,
    });
  }
}
```

```typescript
/**
 * WORK_IN Controller example
 */

@Controller('WORK_IN')
export class WorkInController {
  constructor(
    private readonly personService: PersonService,
    private readonly workInService: WorkInService,
    private readonly companyService: CompanyService,
  ) {}

  @Post('/:from/:to')
  async workIn(
    @Param('from') from: string,
    @Param('to') to: string,
    @Body() workInDto: WorkInDto,
  ): Promise<WorkInDto[]> {
    return this.workInService.create(
      workInDto,
      { name: from },
      { name: to },
      this.personService,
      this.companyService,
    );
  }

  @Get()
  async findAll(): Promise<[PersonDto, WorkInDto, CompanyDto][]> {
    return this.workInService.findAll();
  }
}
```

## License

[MIT licensed](LICENSE).
