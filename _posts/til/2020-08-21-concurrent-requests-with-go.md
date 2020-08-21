---
layout: post
title: Making concurrent HTTP requests in Go
author: Javier Garcia
category: Go
tags: Go, concurrency
---

Today I didn't learn about channels, but I did have to code a function which
would make a bunch of requests concurrently to minimise execution time. It's
nothing out of this world but I did like how it turned up, so just sharing that
over here :)

```go
func (c APIClient) findProducts(productCodes []string) ([]*product, error) {
	wg := sync.WaitGroup{}
	wg.Add(len(productCodes))

	ch := make(chan *product)
	errCh := make(chan error)

	for _, code := range productCodes {
		go func(code string, ch chan<- *product, errCh chan<- error) {
			defer wg.Done()

			p, err := c.findProduct(code)
			if err != nil {
				errCh <- err
			} else {
				ch <- p
			}
		}(code, ch, errCh)
	}

	wg.Wait()

	close(ch)
	close(errCh)

	var products []*product
	for p := range ch {
		products = append(products, p)
	}

  // I'm not too sure about this mechanic... but YOLO.
	var err error
	for e := range errCh {
		err = errors.Wrap(err, e.Error())
	}

	return products, err
}
```
