# Microservice

### O que são Microserviços?

Microserviços são uma arquitetura de software onde uma aplicação é dividida em pequenos serviços independentes, cada um responsável por uma funcionalidade específica. Esses serviços comunicam-se entre si, mas podem ser desenvolvidos, implantados e escalados de forma independente. Essa arquitetura é diferente da abordagem monolítica, onde toda a aplicação é criada em um único código-base.

#### Principais Vantagens de Microserviços:
1. **Escalabilidade Independente**: Cada serviço pode escalar conforme a necessidade.
2. **Manutenção e Atualização Simples**: Facilita o desenvolvimento e a atualização contínua.
3. **Divisão de Responsabilidades**: Cada equipe pode se concentrar em um serviço específico.

### Exemplo Prático em NestJS com TypeORM

Vamos criar um exemplo com dois microserviços: um para **Usuários** e outro para **Pedidos**. A ideia é que o serviço de Pedidos se comunique com o de Usuários para obter dados sobre quem fez o pedido.

#### Passo 1: Configuração do Projeto

Primeiro, vamos criar o projeto NestJS e configurar os microserviços. Abra o terminal e execute:

```bash
# Criando projeto principal
nest new microservice-example
cd microservice-example
```

Vamos criar as pastas dos microserviços:

```bash
mkdir user-service
mkdir order-service
```

#### Passo 2: Configuração do `user-service`

1. Dentro da pasta `user-service`, crie o projeto NestJS específico:

   ```bash
   nest new user-service
   ```

2. Instale o **TypeORM** e o **SQLite** (ou outro banco de dados de sua escolha):

   ```bash
   cd user-service
   npm install @nestjs/typeorm typeorm sqlite3
   ```

3. No `user-service/src/app.module.ts`, configure o TypeORM para conectar ao banco de dados e defina uma entidade para o usuário.

   ```typescript
   import { Module } from '@nestjs/common';
   import { TypeOrmModule } from '@nestjs/typeorm';
   import { User } from './user.entity';
   import { UsersService } from './users.service';
   import { UsersController } from './users.controller';

   @Module({
     imports: [
       TypeOrmModule.forRoot({
         type: 'sqlite',
         database: 'users.db',
         entities: [User],
         synchronize: true,
       }),
       TypeOrmModule.forFeature([User]),
     ],
     providers: [UsersService],
     controllers: [UsersController],
   })
   export class AppModule {}
   ```

4. Crie a entidade de usuário (`user.entity.ts`):

   ```typescript
   import { Entity, Column, PrimaryGeneratedColumn } from 'typeorm';

   @Entity()
   export class User {
     @PrimaryGeneratedColumn()
     id: number;

     @Column()
     name: string;

     @Column()
     email: string;
   }
   ```

5. Crie o serviço (`users.service.ts`) e o controlador (`users.controller.ts`) para gerenciar usuários.

   - **`users.service.ts`**: Define os métodos CRUD para os usuários.
   - **`users.controller.ts`**: Define os endpoints para manipular os dados dos usuários.

#### Passo 3: Configuração do `order-service`

1. Dentro da pasta `order-service`, crie outro projeto NestJS:

   ```bash
   nest new order-service
   ```

2. Instale o TypeORM e configure o banco de dados da mesma forma que no serviço de usuários.

3. Defina a entidade de pedido (`order.entity.ts`):

   ```typescript
   import { Entity, Column, PrimaryGeneratedColumn } from 'typeorm';

   @Entity()
   export class Order {
     @PrimaryGeneratedColumn()
     id: number;

     @Column()
     product: string;

     @Column()
     userId: number; // ID do usuário que fez o pedido
   }
   ```

4. No `order-service`, vamos criar um cliente de microserviço para o serviço de usuários. Isso permitirá que o serviço de pedidos faça chamadas ao serviço de usuários para buscar informações adicionais.

#### Passo 4: Comunicação Entre os Microserviços

1. Para que os microserviços possam se comunicar, vamos utilizar o **TCP**.

2. No `user-service/src/main.ts`, configure o serviço para escutar em uma porta TCP:

   ```typescript
   import { NestFactory } from '@nestjs/core';
   import { AppModule } from './app.module';
   import { Transport, MicroserviceOptions } from '@nestjs/microservices';

   async function bootstrap() {
     const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
       transport: Transport.TCP,
       options: {
         port: 3001,
       },
     });
     await app.listen();
   }
   bootstrap();
   ```

3. No `order-service`, configure o cliente para se comunicar com o `user-service`:

   ```typescript
   import { ClientProxyFactory, Transport, ClientProxy } from '@nestjs/microservices';
   import { Injectable } from '@nestjs/common';

   @Injectable()
   export class UsersClient {
     private client: ClientProxy;

     constructor() {
       this.client = ClientProxyFactory.create({
         transport: Transport.TCP,
         options: {
           port: 3001, // Porta do user-service
         },
       });
     }

     // Função para buscar um usuário pelo ID
     getUser(userId: number) {
       return this.client.send({ cmd: 'get-user' }, userId);
     }
   }
   ```

4. No `user-service`, implemente o handler para responder à requisição do `order-service`:

   ```typescript
   import { Controller } from '@nestjs/common';
   import { MessagePattern } from '@nestjs/microservices';
   import { UsersService } from './users.service';

   @Controller()
   export class UsersController {
     constructor(private readonly usersService: UsersService) {}

     @MessagePattern({ cmd: 'get-user' })
     async getUser(id: number) {
       return this.usersService.findOne(id);
     }
   }
   ```

5. Agora, quando o `order-service` precisar de informações sobre um usuário, ele pode usar o cliente `UsersClient` para enviar uma requisição ao `user-service`.

#### Passo 5: Finalizando o Pedido no `order-service`

No `order-service`, ao criar um pedido, buscamos o usuário para garantir que ele existe:

```typescript
import { Injectable } from '@nestjs/common';
import { OrdersService } from './orders.service';
import { UsersClient } from './users.client';

@Injectable()
export class OrderController {
  constructor(
    private readonly ordersService: OrdersService,
    private readonly usersClient: UsersClient
  ) {}

  async createOrder(userId: number, product: string) {
    const user = await this.usersClient.getUser(userId).toPromise();
    if (!user) throw new Error('User not found');
    
    return this.ordersService.create({ userId, product });
  }
}
```

### Resumo

1. Criamos dois microserviços: `user-service` e `order-service`.
2. Configuramos cada um com **TypeORM** para se conectar ao banco de dados.
3. Implementamos comunicação entre eles usando **TCP**.
4. No `order-service`, verificamos os dados de um usuário no `user-service` antes de processar um pedido.

Esse é um exemplo básico, mas ele já demonstra os conceitos principais de microserviços e como implementá-los em **NestJS** com **TypeORM**.
