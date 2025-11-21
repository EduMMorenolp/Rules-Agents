# AccessAI Backend - Stream Processing Rules

## 🎥 Camera Stream Configuration

### Camera Stream Setup
```javascript
// services/cameraService.js
import cv from 'opencv4nodejs';
import axios from 'axios';

class CameraService {
    constructor() {
        this.activeStreams = new Map();
        this.aiServiceUrl = process.env.AI_SERVICE_URL || 'http://localhost:8000';
    }

    async initializeCamera(deviceId, rtspUrl) {
        try {
            const cap = new cv.VideoCapture(rtspUrl || 0);
            this.activeStreams.set(deviceId, cap);
            return cap;
        } catch (error) {
            throw new Error(`Failed to initialize camera: ${error.message}`);
        }
    }
```

### Upload Function Pattern
```javascript
// middlewares/uploadToS3.js
const uploadToS3 = async (folder, base64File, fileName) => {
    try {
        // Validar que el archivo base64 sea válido
        if (!base64File || typeof base64File !== 'string') {
            throw new Error('Invalid file data');
        }

        // Convertir base64 a buffer
        const fileBuffer = Buffer.from(base64File, 'base64');
        
        // Detectar tipo de archivo
        const fileType = await FileType.fromBuffer(fileBuffer);
        if (!fileType) {
            throw new Error('Could not determine file type');
        }

        // Validar tipo de archivo
        const allowedTypes = ['image/jpeg', 'image/png', 'image/webp', 'application/pdf'];
        if (!allowedTypes.includes(fileType.mime)) {
            throw new Error(`File type ${fileType.mime} not allowed`);
        }

        // Validar tamaño (100MB máximo)
        const maxSize = 100 * 1024 * 1024; // 100MB
        if (fileBuffer.length > maxSize) {
            throw new Error('File size exceeds 100MB limit');
        }

        // Generar key única
        const timestamp = Date.now();
        const key = `${folder}/${fileName}-${timestamp}.${fileType.ext}`;

        // Upload a S3
        const command = new PutObjectCommand({
            Bucket: BUCKET_NAME,
            Key: key,
            Body: fileBuffer,
            ContentType: fileType.mime,
            // Hacer público para lectura
            ACL: 'public-read'
        });

        await s3Client.send(command);

        // Retornar URL pública
        return `https://${BUCKET_NAME}.s3.amazonaws.com/${key}`;
    } catch (error) {
        console.error('S3 Upload Error:', error);
        throw new Error(`Upload failed: ${error.message}`);
    }
};

export default uploadToS3;
```

## 🔍 File Validation

### File Type Validation - Según Postman Collection
```javascript
// validators/fileValidators.js
export const FILE_VALIDATIONS = {
    avatarFile: {
        types: ['image/jpeg', 'image/png', 'image/webp'],
        maxSize: 100 * 1024 * 1024, // 100MB
        folder: 'avatars',
        endpoint: 'PATCH /profile/me/avatar'
    },
    bannerFile: {
        types: ['image/jpeg', 'image/png', 'image/webp'],
        maxSize: 100 * 1024 * 1024, // 100MB
        folder: 'banners',
        endpoint: 'PATCH /profile/me/banner'
    },
    logoFile: {
        types: ['image/jpeg', 'image/png', 'image/webp'],
        maxSize: 100 * 1024 * 1024, // 100MB
        folder: 'logos',
        endpoint: 'POST /auth/register (empresas)'
    },
    statuteFile: {
        types: ['application/pdf'],
        maxSize: 100 * 1024 * 1024, // 100MB
        folder: 'statutes',
        endpoint: 'POST /auth/register (empresas)'
    },
    mediaFile: {
        types: ['image/jpeg', 'image/png', 'image/webp'],
        maxSize: 100 * 1024 * 1024, // 100MB
        folder: 'posts',
        endpoint: 'POST /posts'
    }
};

export const validateFileType = (fileType, category) => {
    const validation = FILE_VALIDATIONS[category];
    if (!validation) {
        throw new Error('Invalid file category');
    }
    
    if (!validation.types.includes(fileType)) {
        throw new Error(`File type ${fileType} not allowed for ${category}`);
    }
    
    return true;
};

export const validateFileSize = (fileSize, category) => {
    const validation = FILE_VALIDATIONS[category];
    if (!validation) {
        throw new Error('Invalid file category');
    }
    
    if (fileSize > validation.maxSize) {
        const maxSizeMB = Math.round(validation.maxSize / 1024 / 1024);
        throw new Error(`File size exceeds ${maxSizeMB}MB limit for ${category}`);
    }
    
    return true;
};
```

### Base64 Validation
```javascript
// validators/fileValidators.js
export const validateBase64 = (base64String) => {
    if (!base64String || typeof base64String !== 'string') {
        throw new Error('Invalid base64 string');
    }
    
    // Verificar formato base64
    const base64Regex = /^[A-Za-z0-9+/]*={0,2}$/;
    if (!base64Regex.test(base64String)) {
        throw new Error('Invalid base64 format');
    }
    
    // Verificar que no esté vacío
    if (base64String.length === 0) {
        throw new Error('Empty file data');
    }
    
    return true;
};

export const extractBase64Data = (dataUrl) => {
    // Remover data:image/jpeg;base64, si existe
    if (dataUrl.startsWith('data:')) {
        const base64Data = dataUrl.split(',')[1];
        if (!base64Data) {
            throw new Error('Invalid data URL format');
        }
        return base64Data;
    }
    
    return dataUrl;
};
```

## 🖼️ Avatar Upload Pattern

### Avatar/Banner Controllers - Según Endpoints Reales
```javascript
// controllers/profileController.js
import uploadToS3 from '../middlewares/uploadToS3.js';
import { validateBase64, extractBase64Data } from '../validators/fileValidators.js';

class ProfileController {
    // PATCH /profile/me/avatar
    changeAvatar = asyncHandler(async (req, res) => {
        const { avatarFile } = req.body;
        const userId = req.user.userId;

        // Validar que se envió archivo
        if (!avatarFile) {
            return res.status(422).json({
                error: 'Validation Error',
                message: 'Avatar file is required'
            });
        }

        // Extraer y validar base64
        const base64Data = extractBase64Data(avatarFile);
        validateBase64(base64Data);

        // Upload a S3
        const avatarUrl = await uploadToS3('avatars', base64Data, `${userId}-avatar`);

        // Actualizar usuario en BD
        await User.update(
            { avatarUrl },
            { where: { id: userId } }
        );

        res.status(200).json({
            message: 'Avatar updated successfully',
            avatarUrl
        });
    });

    // PATCH /profile/me/banner
    changeBanner = asyncHandler(async (req, res) => {
        const { bannerFile } = req.body;
        const userId = req.user.userId;

        // Validar que se envió archivo
        if (!bannerFile) {
            return res.status(422).json({
                error: 'Validation Error',
                message: 'Banner file is required'
            });
        }

        // Extraer y validar base64
        const base64Data = extractBase64Data(bannerFile);
        validateBase64(base64Data);

        // Upload a S3
        const bannerUrl = await uploadToS3('banners', base64Data, `${userId}-banner`);

        // Actualizar usuario en BD
        await User.update(
            { bannerUrl },
            { where: { id: userId } }
        );

        res.status(200).json({
            message: 'Banner updated successfully',
            bannerUrl
        });
    });
}
```

## 📄 Document Upload Pattern

### Statute Upload (Business Profiles)
```javascript
// services/authService.js - Durante registro de empresa
async register(userData) {
    const { logoFile, statuteFile, businessName } = userData;
    
    // Crear usuario primero
    const user = await User.create({
        email: userData.email.toLowerCase().trim(),
        passwordHash: userData.password,
        firstName: businessName,
        roleId: role.id
    });

    let avatarUrl = null;
    let statuteUrl = null;

    try {
        // Upload logo si se proporciona
        if (logoFile) {
            const base64Data = extractBase64Data(logoFile);
            validateBase64(base64Data);
            avatarUrl = await uploadToS3('logos', base64Data, `${user.id}-logo`);
            await user.update({ avatarUrl });
        }

        // Upload estatuto si se proporciona
        if (statuteFile) {
            const base64Data = extractBase64Data(statuteFile);
            validateBase64(base64Data);
            statuteUrl = await uploadToS3('statutes', base64Data, `${user.id}-statute`);
        }

        // Crear perfil de negocio
        await BusinessProfile.create({
            userId: user.id,
            businessName,
            representative: userData.representative,
            statuteUrl
        });

        return user;
    } catch (error) {
        // Si falla el upload, eliminar usuario para mantener consistencia
        await user.destroy();
        throw new Error(`File upload failed: ${error.message}`);
    }
}
```

## 🎨 Post Media Upload

### Post with Media - Endpoint POST /posts
```javascript
// controllers/postController.js
class PostController {
    createPost = asyncHandler(async (req, res) => {
        const { content, mediaFile, visibility = 'public' } = req.body;
        const userId = req.user.userId;

        let mediaUrl = null;

        // Validar que hay contenido o media
        if (!content && !mediaFile) {
            return res.status(422).json({
                error: 'Validation Error',
                message: 'Content or media file is required'
            });
        }

        // Upload media si se proporciona
        if (mediaFile) {
            const base64Data = extractBase64Data(mediaFile);
            validateBase64(base64Data);
            
            // Generar nombre único para el post
            const timestamp = Date.now();
            mediaUrl = await uploadToS3('posts', base64Data, `${userId}-post-${timestamp}`);
        }

        // Crear post
        const post = await Post.create({
            userId,
            content,
            mediaUrl,
            visibility
        });

        // Retornar post con datos del autor
        const postWithAuthor = await Post.findByPk(post.id, {
            include: [{
                model: User,
                as: 'author',
                attributes: ['id', 'firstName', 'lastName', 'avatarUrl']
            }]
        });

        res.status(201).json(postWithAuthor);
    });
}
```

## 🗑️ File Cleanup

### Cleanup Old Files
```javascript
// utils/fileCleanup.js
import { S3Client, DeleteObjectCommand } from '@aws-sdk/client-s3';

const s3Client = new S3Client({
    region: process.env.AWS_REGION || 'us-east-1',
    credentials: {
        accessKeyId: process.env.AWS_ACCESS_KEY_ID,
        secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY
    }
});

export const deleteFromS3 = async (fileUrl) => {
    try {
        // Extraer key de la URL
        const url = new URL(fileUrl);
        const key = url.pathname.substring(1); // Remover '/' inicial

        const command = new DeleteObjectCommand({
            Bucket: process.env.AWS_BUCKET_NAME,
            Key: key
        });

        await s3Client.send(command);
        console.log(`File deleted from S3: ${key}`);
    } catch (error) {
        console.error('S3 Delete Error:', error);
        // No lanzar error, solo loggear
    }
};

// Cleanup cuando se actualiza avatar
export const cleanupOldAvatar = async (oldAvatarUrl) => {
    if (oldAvatarUrl && oldAvatarUrl.includes(process.env.AWS_BUCKET_NAME)) {
        await deleteFromS3(oldAvatarUrl);
    }
};
```

## 🔧 Middleware for File Uploads

### File Upload Middleware
```javascript
// middlewares/fileUpload.js
export const validateFileUpload = (category) => {
    return (req, res, next) => {
        const fileField = getFileFieldName(category);
        const fileData = req.body[fileField];

        if (!fileData) {
            return next(); // Archivo opcional
        }

        try {
            // Validar formato base64
            const base64Data = extractBase64Data(fileData);
            validateBase64(base64Data);

            // Validar tamaño aproximado (base64 es ~33% más grande)
            const approximateSize = (base64Data.length * 3) / 4;
            validateFileSize(approximateSize, category);

            next();
        } catch (error) {
            return res.status(422).json({
                error: 'Validation Error',
                message: error.message
            });
        }
    };
};

const getFileFieldName = (category) => {
    const fieldNames = {
        avatars: 'avatarFile',
        logos: 'logoFile',
        banners: 'bannerFile',
        statutes: 'statuteFile',
        posts: 'mediaFile'
    };
    
    return fieldNames[category] || 'file';
};
```

## 📊 File Upload Monitoring

### Upload Metrics
```javascript
// utils/uploadMetrics.js
export const logUploadMetrics = (category, fileSize, duration, success = true) => {
    const metrics = {
        timestamp: new Date().toISOString(),
        category,
        fileSize: Math.round(fileSize / 1024), // KB
        duration: `${duration}ms`,
        success,
        type: 'FILE_UPLOAD'
    };
    
    console.log(metrics);
    
    // En producción, enviar a servicio de métricas
    if (process.env.NODE_ENV === 'production') {
        // sendToMetricsService(metrics);
    }
};

// Usar en uploadToS3
const uploadToS3 = async (folder, base64File, fileName) => {
    const startTime = Date.now();
    const fileSize = Buffer.from(base64File, 'base64').length;
    
    try {
        const result = await performUpload(folder, base64File, fileName);
        
        logUploadMetrics(folder, fileSize, Date.now() - startTime, true);
        return result;
    } catch (error) {
        logUploadMetrics(folder, fileSize, Date.now() - startTime, false);
        throw error;
    }
};
```

Esta configuración de stream processing garantiza procesamiento eficiente de video en tiempo real para AccessAI.