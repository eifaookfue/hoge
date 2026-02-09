version: 0.2

phases:
  install:
    commands:
      - echo "=== Check Java ==="
      - java -version
      - echo "=== Check Maven ==="
      - which mvn
      - mvn -version
