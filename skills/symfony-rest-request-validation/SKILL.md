---
name: symfony-rest-request-validation
description: Implement or refactor Symfony REST request handling so JSON bodies are deserialized into request DTOs with Symfony Serializer, validated with Symfony Validator constraints, and applied explicitly to domain entities without unsafe mass assignment. Use for PATCH/POST endpoints, request DTOs, validation errors, allowed-field enforcement, and avoiding manual json_decode validation in controllers or services.
---

# Symfony Rest Request Validation

## Purpose

Use this skill when implementing or refactoring Symfony REST endpoints that accept JSON request bodies.

The preferred pattern is:

1. Controller, controller helper, or request-boundary resolver receives the HTTP request payload.
2. The request boundary deserializes JSON into a request DTO, not a Doctrine entity.
3. Symfony Serializer rejects unsupported input fields.
4. Symfony Validator checks DTO constraints.
5. The application service receives the validated DTO or command, not raw HTTP input.
6. The service applies only explicitly supported DTO fields to the loaded entity.
7. Errors are converted to the project's existing error response format and HTTP statuses.

## Baseline Rules

- Check the workspace `.aiignore` before reading or changing project files.
- Follow the existing controller, service, serializer, validator, and error-response style.
- Keep controllers thin.
- Treat JSON body parsing and request DTO validation as controller/request-boundary work. Use a small shared resolver/helper when several controllers need the same pattern.
- Do not pass raw request payloads, Symfony `Request` objects, or HTTP-specific parsing concerns into ordinary application or domain services unless the class is explicitly a request-boundary handler.
- Do not deserialize request JSON directly into Doctrine entities for mutable endpoints.
- Do not use broad mass assignment from request data into entities.
- Do not manually parse and validate JSON fields with `json_decode`, `array_key_exists`, `is_int`, or whitelist arrays when Symfony Serializer and Validator can model the same contract.
- Do not create DTOs only for read responses when the project already uses serializer groups on entities for output.
- For `PATCH`, allow only intentionally supported fields and apply each field explicitly.
- Preserve existing behavior unless the user approved a behavior change.

## Request DTO Pattern

Create a small DTO in the feature namespace near the service or controller using it.

Example:

```php
use Symfony\Component\Validator\Constraints as Assert;
use Symfony\Component\Validator\Context\ExecutionContextInterface;

final class CartItemUpdateRequest
{
    private bool $quantityProvided = false;

    #[Assert\Type("integer")]
    #[Assert\Positive]
    private ?int $quantity = null;

    public function getQuantity(): ?int
    {
        return $this->quantity;
    }

    public function setQuantity(?int $quantity): void
    {
        $this->quantityProvided = true;
        $this->quantity = $quantity;
    }

    public function hasQuantity(): bool
    {
        return $this->quantityProvided;
    }

    #[Assert\Callback]
    public function validate(ExecutionContextInterface $context): void
    {
        if (!$this->quantityProvided) {
            $context->buildViolation("Request body must contain quantity.")
                ->addViolation();
        }
    }
}
```

Use `has<Field>()` flags for PATCH DTOs when the endpoint needs to distinguish:

- field absent;
- field present with `null`;
- field present with a concrete value.

## Deserialization

Deserialize through `SerializerInterface` at the controller/request boundary and reject extra fields:

```php
use Symfony\Component\Serializer\Exception\ExceptionInterface as SerializerExceptionInterface;
use Symfony\Component\Serializer\Normalizer\AbstractNormalizer;

try {
    $dto = $this->serializer->deserialize($payload, CartItemUpdateRequest::class, "json", [
        AbstractNormalizer::ALLOW_EXTRA_ATTRIBUTES => false,
    ]);
} catch (SerializerExceptionInterface $exception) {
    throw new \InvalidArgumentException($exception->getMessage(), previous: $exception);
}
```

Keep a small empty-body guard when the endpoint requires a body:

```php
if (trim($payload) === "") {
    throw new \InvalidArgumentException("Request body must contain valid JSON.");
}
```

This guard is acceptable because it checks request shape before deserialization; it is not field-level business validation.

When the same deserialization, unsupported-field rejection, and validation pattern appears in multiple controllers, extract it into a shared request-boundary resolver. Keep that resolver focused on translating raw HTTP payloads into validated DTOs; keep business application services focused on domain loading, authorization/ownership checks, state changes, and persistence.

## Validation

Validate the DTO through `ValidatorInterface`:

```php
$violations = $this->validator->validate($dto);

if (count($violations) > 0) {
    throw new \InvalidArgumentException((string) $violations);
}
```

Prefer built-in constraints before custom checks:

- `#[Assert\NotNull]` for required non-null values.
- `#[Assert\Type("integer")]` for integer input.
- `#[Assert\Positive]` for values greater than zero.
- `#[Assert\PositiveOrZero]` for values greater than or equal to zero.
- `#[Assert\Choice]`, `#[Assert\Range]`, `#[Assert\Length]`, and similar constraints where they match the contract.
- `#[Assert\Callback]` for cross-field rules, PATCH "at least one field" rules, and presence tracking.

## Applying Changes

After validation, pass the DTO or command into the application service and update entities explicitly:

```php
if ($dto->hasQuantity() && $dto->getQuantity() !== null) {
    $item->setQuantity($dto->getQuantity());
}
```

For owner-scoped resources, load with ownership and state conditions in the repository before applying changes. Do not load by id first and check ownership afterward.

## Error Handling

Use the project's existing error mapper, exception listener, controller helper, or test convention.

Typical status mapping:

- `400 Bad Request` for malformed JSON, unsupported fields, or invalid request shape when that is the project's convention.
- `422 Unprocessable Entity` for validation errors if the project already uses it.
- `404 Not Found` for missing, foreign, or inaccessible owner-scoped resources when the API contract requires indistinguishable responses.

Do not introduce a second error envelope unless the project already has no convention.

## OpenAPI

When request handling changes the API contract, update OpenAPI attributes/config in the same implementation step.

Document:

- request body schema;
- allowed fields;
- required headers and query/path parameters;
- success responses;
- validation and bad-request responses;
- owner-scoped `404` behavior when relevant.

## Tests

Cover at least:

- valid request body;
- malformed JSON;
- empty body when not allowed;
- unsupported fields;
- wrong scalar types;
- boundary constraints such as `quantity <= 0`;
- explicit `null` when it differs from an omitted field;
- owner-scoped foreign-resource denial when relevant.
