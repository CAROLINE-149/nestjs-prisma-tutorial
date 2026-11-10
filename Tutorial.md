# These steps follow the current NestJS docs for Prisma ORM 7 with PostgreSQL.

# 1. Create the NestJS project
bash
npm i -g @nestjs/cli
nest new my-app
cd my-app

# 2. Install Prisma 7 and the driver adapter

The NestJS docs say to install Prisma ORM 7 explicitly. Prisma 7 needs a driver adapter for your database; for PostgreSQL that is @prisma/adapter-pg. 
nestjs

bash
npm install prisma@7 dotenv --save-dev
npm install @prisma/client@7 @prisma/adapter-pg @nestjs/config

# 3. Initialize Prisma
bash
npx prisma init --datasource-provider postgresql --output ../generated/prisma

This creates prisma/schema.prisma, a .env file and prisma.config.ts (new in Prisma 7). Set your connection string in .env:

DATABASE_URL="postgresql://user:password@localhost:5432/mydb?schema=public"

# 4. Configure the schema

In Prisma 7, the connection URL lives in prisma.config.ts, not in schema.prisma. Edit the generator and add a model:

prisma
generator client {
  provider     = "prisma-client"
  output       = "../generated/prisma"
  moduleFormat = "cjs"
}

datasource db {
  provider = "postgresql"
}

model User {
  id    Int     @id @default(autoincrement())
  email String  @unique
  name  String?
}

The moduleFormat = "cjs" line matters. Prisma v7 ships as an ES module by default, which doesn’t work with NestJS’s CommonJS setup, so setting it to cjs makes Prisma generate a CommonJS module. 
dev

# 5. Run the migration and generate the client
bash
npx prisma migrate dev --name init
npx prisma generate

# 6. Register the config module

In app.module.ts:

ts
import { ConfigModule } from '@nestjs/config';

@Module({
  imports: [ConfigModule.forRoot({ isGlobal: true }), PrismaModule],
})
export class AppModule {}

# 7. Create the Prisma module and service
bash
nest g module prisma
nest g service prisma

src/prisma/prisma.service.ts:

ts
import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { PrismaPg } from '@prisma/adapter-pg';
import { PrismaClient } from '../../generated/prisma/client';

@Injectable()
export class PrismaService
  extends PrismaClient
  implements OnModuleInit, OnModuleDestroy
{
  constructor(config: ConfigService) {
    super({
      adapter: new PrismaPg({
        connectionString: config.getOrThrow<string>('DATABASE_URL'),
      }),
    });
  }

  async onModuleInit() {
    await this.$connect();
  }

  async onModuleDestroy() {
    await this.$disconnect();
  }
}

Export it from prisma.module.ts so other modules can use it:

ts
@Global()
@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class PrismaModule {}
8. Use it in a service
ts
@Injectable()
export class UsersService {
  constructor(private readonly prisma: PrismaService) {}

  findAll() {
    return this.prisma.user.findMany();
  }

  create(email: string, name?: string) {
    return this.prisma.user.create({ data: { email, name } });
  }
}

# 9. Run it
bash
npm run start:dev

# Notes
Add generated/ to .gitignore and run npx prisma generate in your build or CI step.
After each schema change, run npx prisma migrate dev.
For production, use npx prisma migrate deploy.
For MySQL or SQL Server, swap the adapter package: @prisma/adapter-mariadb for MySQL/MariaDB, @prisma/adapter-mssql for SQL Server. 
nestjs