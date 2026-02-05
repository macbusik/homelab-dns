1. IaC: Cała infrastruktura musi być zdefiniowana w Terraform (OpenTofu jest akceptowalne).
2. Stan: Używaj lokalnego backendu Terraform do testów, S3 do produkcji.
3. Symulacja: Do testów lokalnych wykorzystuj LocalStack. Upewnij się, że provider AWS w Terraform ma ustawione odpowiednie endpoints na http://localhost:4566.
4. Bezpieczeństwo: Nigdy nie hardkoduj AWS_ACCESS_KEY_ID ani AWS_SECRET_ACCESS_KEY w plikach .tf. Używaj zmiennych.

