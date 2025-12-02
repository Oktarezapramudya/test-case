# Test-case

## Deskripsi
Repository ini berisi contoh aplikasi sederhana menggunakan **Golang** dengan framework **Gin**, yang di-deploy ke **Google Cloud Run** menggunakan **Docker** dan otomatisasi **CI/CD dengan GitHub Actions**.

---

## Membuat Program

```go
package main

import "github.com/gin-gonic/gin"

func main() {
	route := gin.Default()

	route.GET("/", func(ctx *gin.Context) {
		ctx.JSON(200, gin.H{
			"msg": "Hello World",
		})
	})
	
	route.Run(":8080")
}
```

Jalankan perintah berikut untuk dilakukan pengujian
```go
go run main.go
```

![image alt](https://github.com/Oktarezapramudya/test-case/blob/main/Screenshot%202025-12-02%20at%2017.11.52.png?raw=true)
![image alt](https://github.com/Oktarezapramudya/test-case/blob/main/Screenshot%202025-12-02%20at%2017.11.43.png?raw=true)


## Membuat Dockerfile

```Dockerfile
FROM golang:1.25 AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o main .

FROM gcr.io/distroless/base-debian12

WORKDIR /app
COPY --from=builder /app/main .

EXPOSE 8080

CMD ["./main"]

```

Selanjutnya menjalankan perintah berikut, untuk menjalankan proses build dan membentuk docker image
```docker
docker build -t test-case
```

Selanjutnya menjalankan perintah berikut untuk dapat menggunakan Google Cloud SDK
```gcloud
gcloud auth login
```
Selanjutnya melakukan docker pull untuk menyimpan docker images ke artifact registry

Selanjutnya dilakukan deployment ke layanan cloud run dan dilanjutkan dengan pengujian menggunakan Postman

![image alt](https://github.com/Oktarezapramudya/test-case/blob/main/Screenshot%202025-12-02%20at%2023.57.01.png?raw=true)

Selanjutnya untuk menggunakan Github Action, membuat file .yml
```yml
name: CI/CD to Cloud Run

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Authenticate to GCP
        uses: google-github-actions/auth@v1
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}

      - name: Set up gcloud
        uses: google-github-actions/setup-gcloud@v1
        with:
          project_id: ${{ secrets.GCP_PROJECT_ID }}
          version: 'latest'

      - name: Configure Docker for Artifact Registry
        run: gcloud auth configure-docker asia-southeast2-docker.pkg.dev

      - name: Build & Push Docker image
        run: |
          IMAGE="asia-southeast2-docker.pkg.dev/${{ secrets.GCP_PROJECT_ID }}/${{ secrets.ARTIFACT_REPO }}/test-case:latest"
          docker build -t $IMAGE .
          docker push $IMAGE

      - name: Deploy to Cloud Run
        run: |
          IMAGE="asia-southeast2-docker.pkg.dev/${{ secrets.GCP_PROJECT_ID }}/test-case/test-case:latest"
          gcloud run deploy ${{ secrets.CLOUD_RUN_SERVICE }} \
            --image $IMAGE \
            --platform managed \
            --region ${{ secrets.REGION }} \
            --allow-unauthenticated

```

Lalu menjalankan *git push* untuk men-trigger proses auto build dan auto deployment
![image alt](https://github.com/Oktarezapramudya/test-case/blob/main/Screenshot%202025-12-03%20at%2004.08.29.png?raw=true)
