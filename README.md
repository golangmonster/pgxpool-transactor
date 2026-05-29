# pgxpool-transactor

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

pgxpool-transactor is a pgxpool wrapper to execute code fragments within postgres transaction.

Easiest way to get the perfect repository.

## Installation
```bash
go get github.com/golangmonster/pgxpool-transactor/v1
```

## Usage
```go
package main

import (
	"context"
	"fmt"

	"github.com/Masterminds/squirrel"
	"github.com/golangmonster/pgxtransactor"
	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/sirupsen/logrus"
)

type repository struct {
	pool *pgxtransactor.Pool
	pgxtransactor.Transactor
}

func New(pool *pgxtransactor.Pool) *repository {
	return &repository{
		pool:       pool,
		Transactor: pool,
	}
}

func (r *repository) CreateUser(ctx context.Context, name string) error {
	qb := squirrel.Insert("users").
		Columns("name").
		Values(name).
		PlaceholderFormat(squirrel.Dollar)

	sql, args, err := qb.ToSql()
	if err != nil {
		return err
	}

	_, err = r.pool.Querier(ctx).Exec(ctx, sql, args...)
	if err != nil {
		return err
	}

	return nil
}

func (r *repository) CreateOutboxItem(ctx context.Context, name string) error {
	qb := squirrel.Insert("user_outbox").
		Columns("name").
		Values(name).
		PlaceholderFormat(squirrel.Dollar)

	sql, args, err := qb.ToSql()
	if err != nil {
		return err
	}

	_, err = r.pool.Querier(ctx).Exec(ctx, sql, args...)
	if err != nil {
		return err
	}

	return nil
}

func main() {
	ctx := context.Background()

	pgPool, err := pgxpool.New(ctx, "dsn")
	if err != nil {
		logrus.Fatal("failed to connect to database ", err)
	}
	defer pgPool.Close()

	if err = pgPool.Ping(ctx); err != nil {
		logrus.Fatal("postgres ping failed ", err)
	}

	pgxTransactor := pgxtransactor.New(pgPool)

	repo := New(pgxTransactor)

	userName := "Anastasia Zemleroyka"

	_ = repo.InTx(ctx, func(ctx context.Context) error {
		err := repo.CreateUser(ctx, userName)
		if err != nil {
			return fmt.Errorf("create user: %w", err)
		}

		err = repo.CreateOutboxItem(ctx, userName)
		if err != nil {
			return fmt.Errorf("create outbox item: %w", err)
		}

		return nil
	})
}

```