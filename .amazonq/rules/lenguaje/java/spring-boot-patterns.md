# Java Spring Boot - Patterns & Best Practices

## ☕ **REGLA OBLIGATORIA: Spring Boot Conventions**

### ✅ **Estructura de Proyecto Maven**
```
src/
├── main/
│   ├── java/
│   │   └── com/company/project/
│   │       ├── ProjectApplication.java
│   │       ├── config/
│   │       ├── controller/
│   │       ├── service/
│   │       ├── repository/
│   │       ├── entity/
│   │       ├── dto/
│   │       └── exception/
│   └── resources/
│       ├── application.yml
│       ├── application-dev.yml
│       └── application-prod.yml
└── test/
    └── java/
        └── com/company/project/
```

## 🏗️ **Entity Patterns**

### **JPA Entity Definition**
```java
// entity/User.java
package com.company.project.entity;

import jakarta.persistence.*;
import jakarta.validation.constraints.*;
import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.AllArgsConstructor;
import org.hibernate.annotations.CreationTimestamp;
import org.hibernate.annotations.UpdateTimestamp;

import java.time.LocalDateTime;
import java.util.List;

@Entity
@Table(name = "users", indexes = {
    @Index(name = "idx_user_email", columnList = "email"),
    @Index(name = "idx_user_active", columnList = "is_active")
})
@Data
@NoArgsConstructor
@AllArgsConstructor
public class User {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "email", nullable = false, unique = true, length = 255)
    @Email(message = "Email debe tener formato válido")
    @NotBlank(message = "Email es requerido")
    private String email;
    
    @Column(name = "password_hash", nullable = false)
    @NotBlank(message = "Password es requerido")
    @Size(min = 8, message = "Password debe tener al menos 8 caracteres")
    private String passwordHash;
    
    @Column(name = "first_name", length = 100)
    @Size(max = 100, message = "Nombre no puede exceder 100 caracteres")
    private String firstName;
    
    @Column(name = "last_name", length = 100)
    @Size(max = 100, message = "Apellido no puede exceder 100 caracteres")
    private String lastName;
    
    @Column(name = "is_active", nullable = false)
    private Boolean isActive = true;
    
    @Column(name = "is_deleted", nullable = false)
    private Boolean isDeleted = false;
    
    @CreationTimestamp
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;
    
    @UpdateTimestamp
    @Column(name = "updated_at", nullable = false)
    private LocalDateTime updatedAt;
    
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<Product> products;
    
    // Método helper para nombre completo
    public String getFullName() {
        return String.format("%s %s", 
            firstName != null ? firstName : "", 
            lastName != null ? lastName : "").trim();
    }
}
```

### **Base Entity Pattern**
```java
// entity/BaseEntity.java
package com.company.project.entity;

import jakarta.persistence.*;
import lombok.Data;
import org.hibernate.annotations.CreationTimestamp;
import org.hibernate.annotations.UpdateTimestamp;

import java.time.LocalDateTime;

@MappedSuperclass
@Data
public abstract class BaseEntity {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "is_active", nullable = false)
    private Boolean isActive = true;
    
    @Column(name = "is_deleted", nullable = false)
    private Boolean isDeleted = false;
    
    @CreationTimestamp
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;
    
    @UpdateTimestamp
    @Column(name = "updated_at", nullable = false)
    private LocalDateTime updatedAt;
}
```

## 🎯 **Repository Patterns**

### **JPA Repository**
```java
// repository/UserRepository.java
package com.company.project.repository;

import com.company.project.entity.User;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.Optional;

@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Método de consulta derivado
    Optional<User> findByEmailAndIsDeletedFalse(String email);
    
    // Consulta personalizada con JPQL
    @Query("SELECT u FROM User u WHERE u.isActive = true AND u.isDeleted = false")
    List<User> findActiveUsers();
    
    // Consulta con paginación
    @Query("SELECT u FROM User u WHERE u.isDeleted = false AND " +
           "(:search IS NULL OR LOWER(u.firstName) LIKE LOWER(CONCAT('%', :search, '%')) OR " +
           "LOWER(u.lastName) LIKE LOWER(CONCAT('%', :search, '%')))")
    Page<User> findUsersWithSearch(@Param("search") String search, Pageable pageable);
    
    // Consulta nativa SQL
    @Query(value = "SELECT COUNT(*) FROM users WHERE created_at >= :startDate", nativeQuery = true)
    Long countUsersCreatedAfter(@Param("startDate") LocalDateTime startDate);
    
    // Soft delete
    @Query("UPDATE User u SET u.isDeleted = true WHERE u.id = :id")
    void softDeleteById(@Param("id") Long id);
}
```

## 🔧 **Service Patterns**

### **Service Layer**
```java
// service/UserService.java
package com.company.project.service;

import com.company.project.dto.UserCreateDTO;
import com.company.project.dto.UserResponseDTO;
import com.company.project.entity.User;
import com.company.project.exception.ResourceNotFoundException;
import com.company.project.exception.DuplicateResourceException;
import com.company.project.repository.UserRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.Optional;

@Service
@RequiredArgsConstructor
@Slf4j
@Transactional(readOnly = true)
public class UserService {
    
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final UserMapper userMapper;
    
    /**
     * Crear nuevo usuario
     */
    @Transactional
    public UserResponseDTO createUser(UserCreateDTO createDTO) {
        log.info("Creando usuario con email: {}", createDTO.getEmail());
        
        // Validar que el email no exista
        if (userRepository.findByEmailAndIsDeletedFalse(createDTO.getEmail()).isPresent()) {
            throw new DuplicateResourceException("Email ya existe: " + createDTO.getEmail());
        }
        
        // Crear entidad
        User user = userMapper.toEntity(createDTO);
        user.setPasswordHash(passwordEncoder.encode(createDTO.getPassword()));
        
        // Guardar
        User savedUser = userRepository.save(user);
        
        log.info("Usuario creado exitosamente con ID: {}", savedUser.getId());
        return userMapper.toResponseDTO(savedUser);
    }
    
    /**
     * Obtener usuario por ID
     */
    public UserResponseDTO getUserById(Long id) {
        User user = userRepository.findById(id)
            .filter(u -> !u.getIsDeleted())
            .orElseThrow(() -> new ResourceNotFoundException("Usuario no encontrado: " + id));
        
        return userMapper.toResponseDTO(user);
    }
    
    /**
     * Listar usuarios con paginación y búsqueda
     */
    public Page<UserResponseDTO> getUsers(String search, Pageable pageable) {
        Page<User> users = userRepository.findUsersWithSearch(search, pageable);
        return users.map(userMapper::toResponseDTO);
    }
    
    /**
     * Actualizar usuario
     */
    @Transactional
    public UserResponseDTO updateUser(Long id, UserCreateDTO updateDTO) {
        User user = userRepository.findById(id)
            .filter(u -> !u.getIsDeleted())
            .orElseThrow(() -> new ResourceNotFoundException("Usuario no encontrado: " + id));
        
        // Validar email único si cambió
        if (!user.getEmail().equals(updateDTO.getEmail())) {
            if (userRepository.findByEmailAndIsDeletedFalse(updateDTO.getEmail()).isPresent()) {
                throw new DuplicateResourceException("Email ya existe: " + updateDTO.getEmail());
            }
        }
        
        // Actualizar campos
        userMapper.updateEntityFromDTO(updateDTO, user);
        
        User updatedUser = userRepository.save(user);
        return userMapper.toResponseDTO(updatedUser);
    }
    
    /**
     * Eliminar usuario (soft delete)
     */
    @Transactional
    public void deleteUser(Long id) {
        User user = userRepository.findById(id)
            .filter(u -> !u.getIsDeleted())
            .orElseThrow(() -> new ResourceNotFoundException("Usuario no encontrado: " + id));
        
        user.setIsDeleted(true);
        user.setIsActive(false);
        userRepository.save(user);
        
        log.info("Usuario eliminado (soft delete): {}", id);
    }
}
```

## 📋 **Controller Patterns**

### **REST Controller**
```java
// controller/UserController.java
package com.company.project.controller;

import com.company.project.dto.UserCreateDTO;
import com.company.project.dto.UserResponseDTO;
import com.company.project.service.UserService;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.web.PageableDefault;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
public class UserController {
    
    private final UserService userService;
    
    /**
     * Crear usuario
     */
    @PostMapping
    public ResponseEntity<UserResponseDTO> createUser(@Valid @RequestBody UserCreateDTO createDTO) {
        UserResponseDTO user = userService.createUser(createDTO);
        return ResponseEntity.status(HttpStatus.CREATED).body(user);
    }
    
    /**
     * Obtener usuario por ID
     */
    @GetMapping("/{id}")
    public ResponseEntity<UserResponseDTO> getUserById(@PathVariable Long id) {
        UserResponseDTO user = userService.getUserById(id);
        return ResponseEntity.ok(user);
    }
    
    /**
     * Listar usuarios con paginación
     */
    @GetMapping
    public ResponseEntity<Page<UserResponseDTO>> getUsers(
            @RequestParam(required = false) String search,
            @PageableDefault(size = 20) Pageable pageable) {
        
        Page<UserResponseDTO> users = userService.getUsers(search, pageable);
        return ResponseEntity.ok(users);
    }
    
    /**
     * Actualizar usuario
     */
    @PutMapping("/{id}")
    public ResponseEntity<UserResponseDTO> updateUser(
            @PathVariable Long id,
            @Valid @RequestBody UserCreateDTO updateDTO) {
        
        UserResponseDTO user = userService.updateUser(id, updateDTO);
        return ResponseEntity.ok(user);
    }
    
    /**
     * Eliminar usuario
     */
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        userService.deleteUser(id);
        return ResponseEntity.noContent().build();
    }
}
```

## 📊 **DTO Patterns**

### **Data Transfer Objects**
```java
// dto/UserCreateDTO.java
package com.company.project.dto;

import jakarta.validation.constraints.*;
import lombok.Data;

@Data
public class UserCreateDTO {
    
    @NotBlank(message = "Email es requerido")
    @Email(message = "Email debe tener formato válido")
    @Size(max = 255, message = "Email no puede exceder 255 caracteres")
    private String email;
    
    @NotBlank(message = "Password es requerido")
    @Size(min = 8, max = 100, message = "Password debe tener entre 8 y 100 caracteres")
    @Pattern(regexp = "^(?=.*[A-Z])(?=.*\\d)(?=.*[@$!%*?&])[A-Za-z\\d@$!%*?&].*$",
             message = "Password debe contener al menos una mayúscula, un número y un carácter especial")
    private String password;
    
    @Size(max = 100, message = "Nombre no puede exceder 100 caracteres")
    private String firstName;
    
    @Size(max = 100, message = "Apellido no puede exceder 100 caracteres")
    private String lastName;
}

// dto/UserResponseDTO.java
package com.company.project.dto;

import lombok.Data;
import java.time.LocalDateTime;

@Data
public class UserResponseDTO {
    private Long id;
    private String email;
    private String firstName;
    private String lastName;
    private String fullName;
    private Boolean isActive;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}
```

## 🔄 **Mapper Patterns**

### **MapStruct Mapper**
```java
// mapper/UserMapper.java
package com.company.project.mapper;

import com.company.project.dto.UserCreateDTO;
import com.company.project.dto.UserResponseDTO;
import com.company.project.entity.User;
import org.mapstruct.*;

@Mapper(componentModel = "spring")
public interface UserMapper {
    
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "passwordHash", ignore = true)
    @Mapping(target = "isActive", constant = "true")
    @Mapping(target = "isDeleted", constant = "false")
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "updatedAt", ignore = true)
    @Mapping(target = "products", ignore = true)
    User toEntity(UserCreateDTO createDTO);
    
    @Mapping(source = "fullName", target = "fullName")
    UserResponseDTO toResponseDTO(User user);
    
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "passwordHash", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "updatedAt", ignore = true)
    @Mapping(target = "products", ignore = true)
    void updateEntityFromDTO(UserCreateDTO updateDTO, @MappingTarget User user);
}
```

## 🔒 **Security Configuration**

### **Spring Security**
```java
// config/SecurityConfig.java
package com.company.project.config;

import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.config.annotation.authentication.configuration.AuthenticationConfiguration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);
    }
    
    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration authConfig) throws Exception {
        return authConfig.getAuthenticationManager();
    }
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(session -> 
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/auth/**").permitAll()
                .requestMatchers("/api/v1/health").permitAll()
                .anyRequest().authenticated()
            );
        
        return http.build();
    }
}
```

## 🧪 **Testing Patterns**

### **Unit Tests**
```java
// UserServiceTest.java
package com.company.project.service;

import com.company.project.dto.UserCreateDTO;
import com.company.project.entity.User;
import com.company.project.exception.DuplicateResourceException;
import com.company.project.repository.UserRepository;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.security.crypto.password.PasswordEncoder;

import java.util.Optional;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    
    @Mock
    private UserRepository userRepository;
    
    @Mock
    private PasswordEncoder passwordEncoder;
    
    @Mock
    private UserMapper userMapper;
    
    @InjectMocks
    private UserService userService;
    
    @Test
    void createUser_Success() {
        // Given
        UserCreateDTO createDTO = new UserCreateDTO();
        createDTO.setEmail("test@example.com");
        createDTO.setPassword("Password123!");
        
        User user = new User();
        user.setEmail("test@example.com");
        
        when(userRepository.findByEmailAndIsDeletedFalse(anyString()))
            .thenReturn(Optional.empty());
        when(userMapper.toEntity(any())).thenReturn(user);
        when(passwordEncoder.encode(anyString())).thenReturn("hashedPassword");
        when(userRepository.save(any())).thenReturn(user);
        
        // When
        userService.createUser(createDTO);
        
        // Then
        verify(userRepository).save(any(User.class));
    }
    
    @Test
    void createUser_DuplicateEmail_ThrowsException() {
        // Given
        UserCreateDTO createDTO = new UserCreateDTO();
        createDTO.setEmail("existing@example.com");
        
        when(userRepository.findByEmailAndIsDeletedFalse(anyString()))
            .thenReturn(Optional.of(new User()));
        
        // When & Then
        assertThrows(DuplicateResourceException.class, 
            () -> userService.createUser(createDTO));
    }
}
```

Esta configuración garantiza desarrollo Spring Boot siguiendo las mejores prácticas de la comunidad Java.