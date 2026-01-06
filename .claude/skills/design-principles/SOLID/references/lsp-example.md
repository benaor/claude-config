# LSP Example

## Before — LSP Violation

```typescript
// Bird.ts
class Bird {
  fly(): void {
    console.log('Flying...');
  }
}

// Penguin.ts — violates LSP
class Penguin extends Bird {
  fly(): void {
    throw new Error('Penguins cannot fly!'); // Breaks substitutability
  }
}

// BirdMigration.ts — consumer breaks with Penguin
function migrateBirds(birds: Bird[]): void {
  birds.forEach(bird => bird.fly()); // Throws for Penguin!
}
```

## After — LSP Applied

```typescript
// Bird.ts — base with common behavior only
interface Bird {
  move(): void;
}

interface FlyingBird extends Bird {
  fly(): void;
}

interface SwimmingBird extends Bird {
  swim(): void;
}

// Sparrow.ts
class Sparrow implements FlyingBird {
  move(): void {
    this.fly();
  }

  fly(): void {
    console.log('Flying...');
  }
}

// Penguin.ts — no LSP violation
class Penguin implements SwimmingBird {
  move(): void {
    this.swim();
  }

  swim(): void {
    console.log('Swimming...');
  }
}

// BirdMigration.ts — works with any Bird
function migrateBirds(birds: Bird[]): void {
  birds.forEach(bird => bird.move()); // Safe for all birds
}
```
