# JavaScript - GraphQL Rules

## 🚀 **REGLA OBLIGATORIA: GraphQL Best Practices**

### ✅ **Schema Definition**
```javascript
// schema/typeDefs.js
import { gql } from 'apollo-server-express';

export const typeDefs = gql`
  type User {
    id: ID!
    email: String!
    firstName: String
    lastName: String
    posts: [Post!]!
    createdAt: String!
  }

  type Post {
    id: ID!
    title: String!
    content: String!
    author: User!
    comments: [Comment!]!
    published: Boolean!
    createdAt: String!
  }

  type Comment {
    id: ID!
    content: String!
    author: User!
    post: Post!
    createdAt: String!
  }

  input CreateUserInput {
    email: String!
    firstName: String
    lastName: String
    password: String!
  }

  input CreatePostInput {
    title: String!
    content: String!
    published: Boolean = false
  }

  type Query {
    users(limit: Int = 10, offset: Int = 0): [User!]!
    user(id: ID!): User
    posts(limit: Int = 10, offset: Int = 0): [Post!]!
    post(id: ID!): Post
  }

  type Mutation {
    createUser(input: CreateUserInput!): User!
    updateUser(id: ID!, input: UpdateUserInput!): User!
    deleteUser(id: ID!): Boolean!
    
    createPost(input: CreatePostInput!): Post!
    updatePost(id: ID!, input: UpdatePostInput!): Post!
    deletePost(id: ID!): Boolean!
  }

  type Subscription {
    postAdded: Post!
    commentAdded(postId: ID!): Comment!
  }
`;
```

## 🔧 **Resolvers Pattern**

### **Query Resolvers**
```javascript
// resolvers/queries.js
import { UserInputError, ForbiddenError } from 'apollo-server-express';

export const queries = {
  users: async (parent, { limit, offset }, { dataSources, user }) => {
    // Verificar autenticación
    if (!user) {
      throw new ForbiddenError('Authentication required');
    }

    return await dataSources.userAPI.getUsers({ limit, offset });
  },

  user: async (parent, { id }, { dataSources, user }) => {
    if (!user) {
      throw new ForbiddenError('Authentication required');
    }

    const userData = await dataSources.userAPI.getUserById(id);
    if (!userData) {
      throw new UserInputError('User not found');
    }

    return userData;
  },

  posts: async (parent, { limit, offset }, { dataSources }) => {
    return await dataSources.postAPI.getPosts({ limit, offset });
  },

  post: async (parent, { id }, { dataSources }) => {
    const post = await dataSources.postAPI.getPostById(id);
    if (!post) {
      throw new UserInputError('Post not found');
    }

    return post;
  }
};
```

### **Mutation Resolvers**
```javascript
// resolvers/mutations.js
import { UserInputError, ForbiddenError } from 'apollo-server-express';
import { validateUserInput, validatePostInput } from '../validators/index.js';

export const mutations = {
  createUser: async (parent, { input }, { dataSources }) => {
    // Validar entrada
    const { errors, isValid } = validateUserInput(input);
    if (!isValid) {
      throw new UserInputError('Invalid input', { validationErrors: errors });
    }

    try {
      return await dataSources.userAPI.createUser(input);
    } catch (error) {
      if (error.message.includes('email already exists')) {
        throw new UserInputError('Email already exists');
      }
      throw error;
    }
  },

  updateUser: async (parent, { id, input }, { dataSources, user }) => {
    if (!user) {
      throw new ForbiddenError('Authentication required');
    }

    // Verificar que el usuario puede actualizar este perfil
    if (user.id !== id && user.role !== 'admin') {
      throw new ForbiddenError('Not authorized to update this user');
    }

    return await dataSources.userAPI.updateUser(id, input);
  },

  createPost: async (parent, { input }, { dataSources, user }) => {
    if (!user) {
      throw new ForbiddenError('Authentication required');
    }

    const { errors, isValid } = validatePostInput(input);
    if (!isValid) {
      throw new UserInputError('Invalid input', { validationErrors: errors });
    }

    return await dataSources.postAPI.createPost({
      ...input,
      authorId: user.id
    });
  },

  deletePost: async (parent, { id }, { dataSources, user }) => {
    if (!user) {
      throw new ForbiddenError('Authentication required');
    }

    const post = await dataSources.postAPI.getPostById(id);
    if (!post) {
      throw new UserInputError('Post not found');
    }

    // Verificar ownership
    if (post.authorId !== user.id && user.role !== 'admin') {
      throw new ForbiddenError('Not authorized to delete this post');
    }

    return await dataSources.postAPI.deletePost(id);
  }
};
```

## 🔗 **Field Resolvers**

### **Nested Data Resolution**
```javascript
// resolvers/fieldResolvers.js
export const fieldResolvers = {
  User: {
    posts: async (parent, args, { dataSources }) => {
      return await dataSources.postAPI.getPostsByUserId(parent.id);
    },

    // Resolver computado
    fullName: (parent) => {
      return `${parent.firstName || ''} ${parent.lastName || ''}`.trim();
    }
  },

  Post: {
    author: async (parent, args, { dataSources }) => {
      return await dataSources.userAPI.getUserById(parent.authorId);
    },

    comments: async (parent, args, { dataSources }) => {
      return await dataSources.commentAPI.getCommentsByPostId(parent.id);
    },

    // Resolver con lógica de negocio
    canEdit: (parent, args, { user }) => {
      if (!user) return false;
      return parent.authorId === user.id || user.role === 'admin';
    }
  },

  Comment: {
    author: async (parent, args, { dataSources }) => {
      return await dataSources.userAPI.getUserById(parent.authorId);
    },

    post: async (parent, args, { dataSources }) => {
      return await dataSources.postAPI.getPostById(parent.postId);
    }
  }
};
```

## 📊 **DataSources Pattern**

### **REST DataSource**
```javascript
// dataSources/userAPI.js
import { RESTDataSource } from 'apollo-datasource-rest';

export class UserAPI extends RESTDataSource {
  constructor() {
    super();
    this.baseURL = process.env.USER_SERVICE_URL;
  }

  willSendRequest(request) {
    // Agregar headers de autenticación
    request.headers.set('Authorization', this.context.token);
    request.headers.set('X-Service-Name', 'graphql-gateway');
  }

  async getUsers({ limit = 10, offset = 0 }) {
    const response = await this.get('users', { limit, offset });
    return response.data;
  }

  async getUserById(id) {
    try {
      return await this.get(`users/${id}`);
    } catch (error) {
      if (error.extensions.response.status === 404) {
        return null;
      }
      throw error;
    }
  }

  async createUser(userData) {
    return await this.post('users', userData);
  }

  async updateUser(id, userData) {
    return await this.put(`users/${id}`, userData);
  }

  async deleteUser(id) {
    await this.delete(`users/${id}`);
    return true;
  }

  // Batch loading para resolver N+1 queries
  async getUsersByIds(ids) {
    const response = await this.get('users/batch', { ids: ids.join(',') });
    return response.data;
  }
}
```

### **Database DataSource**
```javascript
// dataSources/postAPI.js
import { DataSource } from 'apollo-datasource';
import DataLoader from 'dataloader';

export class PostAPI extends DataSource {
  constructor({ db }) {
    super();
    this.db = db;
  }

  initialize(config) {
    this.context = config.context;
    
    // Configurar DataLoaders para batch loading
    this.postLoader = new DataLoader(async (ids) => {
      const posts = await this.db.Post.findAll({
        where: { id: ids },
        order: [['created_at', 'DESC']]
      });
      
      // Mantener orden de IDs solicitados
      return ids.map(id => posts.find(post => post.id === id));
    });

    this.postsByUserLoader = new DataLoader(async (userIds) => {
      const posts = await this.db.Post.findAll({
        where: { authorId: userIds },
        order: [['created_at', 'DESC']]
      });
      
      return userIds.map(userId => 
        posts.filter(post => post.authorId === userId)
      );
    });
  }

  async getPosts({ limit, offset }) {
    return await this.db.Post.findAll({
      limit,
      offset,
      order: [['created_at', 'DESC']]
    });
  }

  async getPostById(id) {
    return await this.postLoader.load(id);
  }

  async getPostsByUserId(userId) {
    return await this.postsByUserLoader.load(userId);
  }

  async createPost(postData) {
    return await this.db.Post.create(postData);
  }

  async updatePost(id, postData) {
    const [updatedRowsCount] = await this.db.Post.update(postData, {
      where: { id }
    });
    
    if (updatedRowsCount === 0) {
      throw new Error('Post not found');
    }
    
    return await this.getPostById(id);
  }

  async deletePost(id) {
    const deletedRowsCount = await this.db.Post.destroy({
      where: { id }
    });
    
    return deletedRowsCount > 0;
  }
}
```

## 🔐 **Authentication & Authorization**

### **Context Setup**
```javascript
// server/context.js
import jwt from 'jsonwebtoken';

export const createContext = async ({ req, connection }) => {
  // WebSocket connection (subscriptions)
  if (connection) {
    return {
      user: connection.context.user,
      dataSources: connection.context.dataSources
    };
  }

  // HTTP request
  let user = null;
  const token = req.headers.authorization?.replace('Bearer ', '');
  
  if (token) {
    try {
      const decoded = jwt.verify(token, process.env.JWT_SECRET);
      user = decoded;
    } catch (error) {
      console.warn('Invalid token:', error.message);
    }
  }

  return {
    user,
    token: req.headers.authorization,
    req
  };
};
```

### **Authorization Directives**
```javascript
// directives/auth.js
import { SchemaDirectiveVisitor } from 'apollo-server-express';
import { ForbiddenError } from 'apollo-server-express';

export class AuthDirective extends SchemaDirectiveVisitor {
  visitFieldDefinition(field) {
    const { resolve = defaultFieldResolver } = field;
    const requiredRole = this.args.requires;

    field.resolve = async function(...args) {
      const [, , context] = args;
      
      if (!context.user) {
        throw new ForbiddenError('Authentication required');
      }

      if (requiredRole && context.user.role !== requiredRole) {
        throw new ForbiddenError(`Requires ${requiredRole} role`);
      }

      return resolve.apply(this, args);
    };
  }
}

// Uso en schema
const typeDefs = gql`
  directive @auth(requires: Role = USER) on FIELD_DEFINITION

  enum Role {
    ADMIN
    USER
  }

  type Mutation {
    deleteUser(id: ID!): Boolean! @auth(requires: ADMIN)
    updateProfile(input: UpdateProfileInput!): User! @auth
  }
`;
```

## 📡 **Subscriptions**

### **Real-time Updates**
```javascript
// resolvers/subscriptions.js
import { PubSub, withFilter } from 'apollo-server-express';

const pubsub = new PubSub();

export const subscriptions = {
  postAdded: {
    subscribe: () => pubsub.asyncIterator(['POST_ADDED'])
  },

  commentAdded: {
    subscribe: withFilter(
      () => pubsub.asyncIterator(['COMMENT_ADDED']),
      (payload, variables) => {
        // Filtrar por postId
        return payload.commentAdded.postId === variables.postId;
      }
    )
  },

  userOnline: {
    subscribe: withFilter(
      () => pubsub.asyncIterator(['USER_STATUS_CHANGED']),
      (payload, variables, context) => {
        // Solo usuarios autenticados
        return !!context.user;
      }
    )
  }
};

// Publicar eventos desde mutations
export const publishPostAdded = (post) => {
  pubsub.publish('POST_ADDED', { postAdded: post });
};

export const publishCommentAdded = (comment) => {
  pubsub.publish('COMMENT_ADDED', { commentAdded: comment });
};
```

## 🔍 **Query Complexity & Rate Limiting**

### **Query Complexity Analysis**
```javascript
// plugins/queryComplexity.js
import { createComplexityLimitRule } from 'graphql-query-complexity';

export const complexityLimitRule = createComplexityLimitRule(1000, {
  maximumComplexity: 1000,
  variables: {},
  createError: (max, actual) => {
    return new Error(`Query complexity ${actual} exceeds maximum ${max}`);
  },
  onComplete: (complexity) => {
    console.log('Query complexity:', complexity);
  }
});

// Configurar en servidor
const server = new ApolloServer({
  typeDefs,
  resolvers,
  validationRules: [complexityLimitRule],
  introspection: process.env.NODE_ENV !== 'production',
  playground: process.env.NODE_ENV !== 'production'
});
```

### **Rate Limiting**
```javascript
// plugins/rateLimiting.js
import { shield, rule, and, or } from 'graphql-shield';
import { RateLimiterMemory } from 'rate-limiter-flexible';

const rateLimiter = new RateLimiterMemory({
  keyGenerator: (root, args, context) => context.user?.id || context.req.ip,
  points: 100, // Número de requests
  duration: 60, // Por minuto
});

const rateLimit = rule({ cache: 'contextual' })(
  async (parent, args, context) => {
    try {
      await rateLimiter.consume(context.user?.id || context.req.ip);
      return true;
    } catch (rejRes) {
      throw new Error('Rate limit exceeded');
    }
  }
);

export const permissions = shield({
  Query: {
    users: rateLimit,
    posts: rateLimit
  },
  Mutation: {
    createPost: and(isAuthenticated, rateLimit),
    createUser: rateLimit
  }
});
```

## 🚀 **Server Setup**

### **Apollo Server Configuration**
```javascript
// server/index.js
import { ApolloServer } from 'apollo-server-express';
import { createServer } from 'http';
import express from 'express';
import { typeDefs } from '../schema/typeDefs.js';
import { resolvers } from '../resolvers/index.js';
import { createContext } from './context.js';
import { UserAPI, PostAPI } from '../dataSources/index.js';

const app = express();

const server = new ApolloServer({
  typeDefs,
  resolvers,
  context: createContext,
  dataSources: () => ({
    userAPI: new UserAPI(),
    postAPI: new PostAPI({ db })
  }),
  subscriptions: {
    path: '/graphql',
    onConnect: async (connectionParams) => {
      // Autenticación para WebSocket
      const token = connectionParams.authorization;
      if (token) {
        try {
          const user = jwt.verify(token.replace('Bearer ', ''), process.env.JWT_SECRET);
          return { user };
        } catch (error) {
          throw new Error('Invalid token');
        }
      }
    }
  },
  plugins: [
    {
      requestDidStart() {
        return {
          willSendResponse(requestContext) {
            console.log('GraphQL Response:', {
              query: requestContext.request.query,
              variables: requestContext.request.variables,
              errors: requestContext.errors
            });
          }
        };
      }
    }
  ]
});

server.applyMiddleware({ app, path: '/graphql' });

const httpServer = createServer(app);
server.installSubscriptionHandlers(httpServer);

export { httpServer, server };
```

Esta configuración proporciona una implementación completa de GraphQL con Node.js siguiendo las mejores prácticas.