# Echo-LSAT

A middleware for [Echo](https://echo.labstack.com/) framework that uses [LSAT](https://lsat.tech/) (a protocol standard for authentication and paid APIs) and provides handler functions to accept microtransactions before serving ad-free content or any paid APIs.

## Installation

Assuming you've installed Go and Echo

1. Run this:

```
go get github.com/getAlby/echo-lsat
```

2. Create `.env` file (refer `.env_example`) and configure `LND_ADDRESS` and `MACAROON_HEX` for LND client or `LNURL_ADDRESS` for LNURL client, `LN_CLIENT_TYPE` (out of LND, LNURL) and `ROOT_KEY` (for minting macaroons).  

## Usage

[This example](https://github.com/getAlby/echo-lsat/blob/main/examples/main.go) shows how to use Echo-LSAT for serving simple JSON response:-

```
package main

import (
	"fmt"
	"log"
	"net/http"
	"os"

	"github.com/getAlby/echo-lsat/echolsat"
	"github.com/getAlby/echo-lsat/ln"

	"github.com/joho/godotenv"
	"github.com/labstack/echo/v4"
)

func main() {
	router := echo.New()

	router.GET("/", func(c echo.Context) error {
		return c.JSON(http.StatusAccepted, map[string]interface{}{
			"code":    http.StatusAccepted,
			"message": "Free content",
		})
	})

	err := godotenv.Load(".env")
	if err != nil {
		log.Fatal("Failed to load .env file")
	}
	lnClient, err := echolsat.InitLnClient(&ln.LNClientConfig{
		LNClientType: os.Getenv("LN_CLIENT_TYPE"),
		LNDConfig: ln.LNDoptions{
			Address:     os.Getenv("LND_ADDRESS"),
			MacaroonHex: os.Getenv("MACAROON_HEX"),
		},
		LNURLConfig: ln.LNURLoptions{
			Address: os.Getenv("LNURL_ADDRESS"),
		},
	})
	if err != nil {
		log.Fatal(err)
	}
	lsatmiddleware, err := echolsat.NewLsatMiddleware(&echolsat.EchoLsatMiddleware{
		Amount:   5,
		LNClient: lnClient,
	})
	if err != nil {
		log.Fatal(err)
	}

	router.Use(lsatmiddleware.Handler)

	router.GET("/protected", func(c echo.Context) error {
		lsatInfo := c.Get("LSAT").(*echolsat.LsatInfo)
		if lsatInfo.Type == echolsat.LSAT_TYPE_FREE {
			return c.JSON(http.StatusAccepted, map[string]interface{}{
				"code":    http.StatusAccepted,
				"message": "Free content",
			})
		} else if lsatInfo.Type == echolsat.LSAT_TYPE_PAID {
			return c.JSON(http.StatusAccepted, map[string]interface{}{
				"code":    http.StatusAccepted,
				"message": "Protected content",
			})
		} else {
			return c.JSON(http.StatusAccepted, map[string]interface{}{
				"code":    http.StatusInternalServerError,
				"message": fmt.Sprint(lsatInfo.Error),
			})
		}
	})

	router.Start("localhost:8080")
}
```

## Testing

Run `go test` to run tests.
