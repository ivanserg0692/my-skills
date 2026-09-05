---
name: hexagonal-architecture
description: >
  Use when structuring a Symfony app with ports & adapters / clean architecture —
  keeping the domain framework-free, defining application input ports and inward-owned
  output ports, and implementing adapters in infrastructure. Use when the task mentions
  hexagonal, ports/adapters, or "keep the domain pure".
---

# Hexagonal Architecture (Ports & Adapters)

> **When to reach for this.** Hexagonal is for complex, long-lived domains where isolating business logic from the framework pays off — rich invariants, multiple delivery mechanisms, logic you expect to outlive the current stack. For typical CRUD and feature work (the ~80% of most apps), use [`layered-architecture`](../layered-architecture) instead; strict ports/adapters there is just overhead. Apply this to the gnarly 20% that earns it, not as a blanket rule.

## The dependency rule

```
        ┌─────────────────────────────────────────┐
        │            Infrastructure                 │  ← Symfony, Doctrine, HTTP, Messenger
        │   (driving + driven adapters)             │
        │   ┌───────────────────────────────────┐   │
        │   │         Application               │   │  ← use cases / handlers
        │   │   ┌───────────────────────────┐   │   │
        │   │   │         Domain            │   │   │  ← entities, value objects, ports
        │   │   │   (no framework imports)  │   │   │
        │   │   └───────────────────────────┘   │   │
        │   └───────────────────────────────────┘   │
        └─────────────────────────────────────────┘
```

**Source code dependencies point inward.** The domain knows nothing about Symfony, Doctrine, or HTTP. Outer layers depend on inner-layer contracts, never the reverse.

There are two directions of interaction at the application boundary:

```text
driving infrastructure adapter -> application input port -> use case
use case -> output port <- driven infrastructure adapter
```

- **Input ports** are owned by the application layer. They describe use cases that driving adapters may invoke. HTTP controllers, Console commands, and Messenger handlers depend on these ports.
- **Output ports** are owned by the innermost layer that needs the external capability, usually the application or domain layer. Infrastructure implements them for Doctrine, Elasticsearch, message brokers, clocks, locks, and other external systems.

Do not introduce an interface for every internal class. Create a port when an operation crosses an architectural boundary or is intentionally exposed as a stable use-case contract.

## Suggested package layout

```
src/
├── Domain/                     # pure PHP — no Symfony/Doctrine "use" statements
│   ├── Order/
│   │   ├── Order.php           # aggregate root (plain object, no #[ORM\...])
│   │   ├── OrderId.php         # value object
│   │   ├── OrderStatus.php     # enum
│   │   └── OrderRepository.php  # OUTPUT PORT owned by the domain
├── Application/                # use cases — orchestrates domain via ports
│   └── Order/
│       ├── CreateOrder.php      # INPUT PORT
│       ├── CreateOrderHandler.php # use-case implementation
│       └── CreateOrderCommand.php # input contract
└── Infrastructure/             # adapters — implement ports using Symfony/Doctrine
    ├── Persistence/Doctrine/
    │   ├── DoctrineOrderRepository.php   # ADAPTER implements Domain\Order\OrderRepository
    │   └── Mapping/Order.orm.xml         # ORM mapping kept OUT of the domain class
    └── Http/
        └── CreateOrderController.php     # driving adapter
```

## Output ports are owned inward

An **output port** is an interface owned by the inner layer that needs an external capability. A repository used directly by the domain belongs in the domain. An external capability needed only to orchestrate a use case may belong in the application layer. Infrastructure implements the port in either case.

```php
// ✅ src/Domain/Order/OrderRepository.php — no framework imports
namespace App\Domain\Order;

interface OrderRepository
{
    public function save(Order $order): void;

    public function ofId(OrderId $id): ?Order;
}
```

```php
// ✅ src/Infrastructure/Persistence/Doctrine/DoctrineOrderRepository.php
namespace App\Infrastructure\Persistence\Doctrine;

use App\Domain\Order\Order;
use App\Domain\Order\OrderId;
use App\Domain\Order\OrderRepository;
use Doctrine\ORM\EntityManagerInterface;

final class DoctrineOrderRepository implements OrderRepository
{
    public function __construct(private readonly EntityManagerInterface $em) {}

    public function save(Order $order): void
    {
        $this->em->persist($order);
        $this->em->flush();
    }

    public function ofId(OrderId $id): ?Order
    {
        return $this->em->find(Order::class, $id->toString());
    }
}
```

Wire the interface to the implementation in `config/services.yaml`:

```yaml
services:
    App\Domain\Order\OrderRepository: '@App\Infrastructure\Persistence\Doctrine\DoctrineOrderRepository'
```

## Keeping the domain framework-free

The hard part in Symfony is that Doctrine attribute mapping (`#[ORM\Entity]`) couples the entity to the ORM. Two acceptable strategies:

1. **Pragmatic (default):** allow Doctrine attributes on the aggregate. You lose "zero framework imports" but keep one class. Fine for most teams.
2. **Strict:** keep the domain class pure and map it with **XML mapping** in `Infrastructure/Persistence/Doctrine/Mapping/`. The domain never imports Doctrine.

```php
// ✅ STRICT — pure aggregate, mapped externally via XML
namespace App\Domain\Order;

final class Order
{
    /** @var Collection<int, OrderLine>|OrderLine[] */
    private array $lines = [];

    private function __construct(
        private readonly OrderId $id,
        private OrderStatus $status,
    ) {}

    public static function place(OrderId $id): self
    {
        return new self($id, OrderStatus::Pending);   // invariant: new orders start Pending
    }
}
```

State the chosen strategy in the project README so the agent stays consistent.

## Application layer = use cases

One handler per use case. It depends only on **ports**, never on adapters.

Define an input port when a driving infrastructure adapter enters the application through that use case:

```php
// ✅ src/Application/Order/CreateOrder.php — INPUT PORT
namespace App\Application\Order;

use App\Domain\Order\OrderId;

interface CreateOrder
{
    public function create(CreateOrderCommand $command): OrderId;
}
```

```php
// ✅ src/Application/Order/CreateOrderHandler.php
namespace App\Application\Order;

use App\Domain\Order\Order;
use App\Domain\Order\OrderId;
use App\Domain\Order\OrderRepository;

final readonly class CreateOrderHandler implements CreateOrder
{
    public function __construct(private OrderRepository $orders) {}

    public function create(CreateOrderCommand $command): OrderId
    {
        $order = Order::place(OrderId::generate());
        $this->orders->save($order);

        return $order->id();
    }
}
```

## Driving adapters

Controllers, Console commands, and Messenger handlers are **driving adapters**. They translate a delivery mechanism into an input-port call and translate the result back out. They contain no business logic and, when the project uses a strict application boundary, depend on the input-port interface rather than a concrete use-case implementation.

```php
// ✅ src/Infrastructure/Http/CreateOrderController.php — driving adapter
namespace App\Infrastructure\Http;

use App\Application\Order\CreateOrder;
use App\Application\Order\CreateOrderCommand;

final readonly class CreateOrderController
{
    public function __construct(private CreateOrder $createOrder) {}

    public function __invoke(CreateOrderCommand $command): string
    {
        return $this->createOrder->create($command)->toString();
    }
}
```

Wire the input port to its application implementation just as output ports are wired to infrastructure adapters:

```yaml
services:
    App\Application\Order\CreateOrder: '@App\Application\Order\CreateOrderHandler'
```

## Data contracts belong to ports

A port must describe not only the operation but also the data contract that crosses its boundary. Do not make an adapter depend on a concrete application DTO when the adapter needs only a smaller, stable capability.

For example, a search-index output adapter may need only a document identifier and its indexable representation. Define that contract beside the output port, then let the application DTO implement it:

```php
// ✅ Owned by the inner output-port boundary
namespace App\Search\Product\Port\Output\Document;

interface ProductSearchIndexDocumentInterface
{
    public function getId(): int;

    /** @return array<string, mixed> */
    public function toArray(): array;
}
```

```php
interface ProductSearchIndexGatewayInterface
{
    /** @param ProductSearchIndexDocumentInterface[] $documents */
    public function bulkIndex(string $indexName, array $documents): BulkIndexResult;
}
```

This keeps the dependency direction explicit:

```text
application DTO -> output-port document contract <- infrastructure adapter
```

- Put the contract in the inner port namespace, not under an Elasticsearch, Doctrine, HTTP, or other infrastructure namespace.
- Type collection elements against the port contract, including PHPDoc generic element types when PHP cannot express them natively.
- Keep adapter-specific metadata and mechanics, such as an Elasticsearch schema version, mapping, index name, or bulk wire format, inside that adapter's infrastructure.
- Prefer the smallest contract the port operation actually needs. Do not expose the entire application DTO merely for adapter convenience.
- Do not create a parallel interface automatically when an immutable DTO is intentionally designed as the stable port contract itself. Introduce a separate interface when it removes a real dependency on a concrete model or supports multiple implementations.

## Gotchas

- Agent puts `EntityManagerInterface` into the application/domain layer — depend on a repository **port** instead.
- Agent imports `Symfony\…` or `Doctrine\…` in `src/Domain/` — the domain must stay framework-free (or use the agreed pragmatic exception for ORM attributes only).
- Agent makes the application layer depend on the concrete `DoctrineOrderRepository` — depend on the interface; wire it in `services.yaml`.
- Agent makes an infrastructure controller, Console command, or Messenger handler depend directly on a concrete application use-case implementation even though the project uses strict input-port boundaries — define an application-owned input port and inject that contract.
- Agent places an input port in infrastructure — the application owns the operations through which driving adapters invoke it.
- Agent assumes every port belongs in the domain — input ports belong in the application, while output ports belong to the innermost layer that needs the capability.
- Agent returns a Symfony `Response` from a use-case handler — return a domain value (ID, DTO); the controller builds the HTTP response.
- Agent types an output port or its collection PHPDoc with a concrete application DTO even though the adapter needs only a smaller contract — define that contract at the port boundary.
- Agent defines a data contract under `Infrastructure` and makes the application implement it — move ownership inward so infrastructure depends on the port.
- Agent puts validation/HTTP concerns in the domain — those belong in adapters.
- Agent generates `#[ORM\Entity]` on a "pure" domain class when the project chose the strict XML-mapping strategy — check the README's stated strategy first.
