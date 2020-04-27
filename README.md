# MANUAL ANGULAR - MIFACT

Este manual tiene la finalidad de estructurar la arquitectura del proyecto Angular de manera que sea sostenible modularizando la aplicacion.

# Creando Modulos

Se creara un Modulo por cada Perfil-Opcion y tantos componentes como opciones del subMenu.
Estos componentes perteneceran al modulo creado.

Para crear un modulo usamos el siguiente comando del Angular CLI o podemos hacerlo de manera manual.
Este comando crea un modulo dentro de la carpeta  `pages` y al añardirle  `--routing` tambien crea un enrutador para rese modulo.
```typescript
ng g module pages/auth --routing
```

Para crear un componente y agregarlo al modulo creado, si no se usa el Angular CLI, se puede hacer manualmente.
```typescript
ng g c pages/auth/login --module auth
```
Si se le agrega el siguiente comando se crearan los componentes sin el test :  `--skipTests`

Al Crear un modulo nos estaria quedando de esta forma.
El archivo `auth.module.ts`
```typescript
import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';

import { AuthRoutingModule } from './auth-routing.module';
import { LoginComponent } from './login/login.component';
import { RouterModule } from '@angular/router';
import { PrimengModule } from 'src/app/primeng/primeng.module';
import { RecaptchaModule } from 'ng-recaptcha';
import { FormsModule, ReactiveFormsModule } from '@angular/forms';


@NgModule({
  imports: [
    CommonModule,
    RouterModule,
    RecaptchaModule,
    FormsModule,
    ReactiveFormsModule,
    PrimengModule,
    AuthRoutingModule
  ],
  declarations: [LoginComponent]
})
export class AuthModule { }
```
El enrutador para este modulo es  `auth-routing.module.ts`.

Aqui se debe importar las rutas y los componentes que se usaran para este modulo.
```typescript
import { NgModule } from '@angular/core';
import { Routes, RouterModule } from '@angular/router';
import { LoginComponent } from './login/login.component';


const routes: Routes = [
    {
        path: '',
        component: LoginComponent
    }
];

@NgModule({
    imports: [RouterModule.forChild(routes)],
    exports: [RouterModule]
})
export class AuthRoutingModule { }
```




# Unir Ruta Principal con Modulos

Se usara la tecnica Lazy Loader Para cargar los modulos.

Luego de Crear los nuevos modulos y sus componetes, tenemos que unirlo con el componente inicial que inicializa la aplicacion, en todo proyecto angular es el app.module y su enrutador app-routing.module.

* `app.module.ts`
* `app-routing.module.ts`

Dentro del enrutador principal es donde vamos a cargar nuestros modulos.
En este caso tenemos 3 Modulos principales.

El modulo para el Admin, que es dashboard del sistema.

El modulo Auth que es publico que contiene los componentes como Login, Recuperar Contraseña y otras vistas que no requieren proteccion.

EL otro modulo es de paginas others-layout que es donde estaran otras paginas como not found, forbian.

En el app-routing.module.ts debemos cargar nuestros modulos.

```typescript
const routes: Routes =[
  {
    path: '',
    redirectTo: 'login',
    pathMatch: 'full',
  }, 
  {
    path: '',
    component: AuthLayoutComponent,
    children: [
      {
        path: 'login',
        loadChildren: () => import('./pages/auth/auth.module').then(m => m.AuthModule)
      }
    ]
  },
  {
    path: '',
    component: AdminLayoutComponent,
    //canActivate: [AuthGuard],
    children: adminRoutes
  },
  {
    path: '',
    component: OthersLayoutComponent,
    canActivate : [OthersGuard],
    children: [
      {
        path: 'others',
        loadChildren: () => import('./pages/others/others.module').then(m => m.OthersModule)
      }
    ]
  },
  {
    path: '**',
    redirectTo: 'login'
  }
];
```

Nuestros modulos o componentes debe ir dentro de childen del component `AuthLayoutComponent`

```typescript
{
    path: '',
    component: AdminLayoutComponent,
    //canActivate: [AuthGuard],
    children: adminRoutes
}
```
En este caso estamos cargando solo componentes en adminRoutes
```typescript
const adminRoutes: Routes = [
  { path: 'contribuyente-mant', component: ContribuyenteMantComponent ,canActivate: [AuthGuard]},
  { path: 'prospectos-mant', component: ProspectosMantComponent ,canActivate: [AuthGuard]},
  { path: 'vehiculo-mant', component: VehiculoMantComponent ,canActivate: [AuthGuard]},
  { path: 'grifo-mant', component: GrifoMantComponent ,canActivate: [AuthGuard]},
];
```



Si hubiesen mas modulos princiales se tendrian que crear de la misma forma y agregarlos al enrutador principal.


# Protegiendo Rutas

Para proteger las rutas se crearan Guard que son quien se encargara de seguridad del lado de Angular.

Se pueden crear varios guards dependiendo que tipo de seguridad o verificacion se requiera para cada modulo o ruta.

En este caso hemos creado un guard para proteger la seguridad del modulo admin-layouth donde estan nuestras rutas.

Este Guard valida al momento de tratar de entrar en cada ruta y verificara si tenemos acceso al recurso solicitado.

Tambien validara si el usuario esta logueado o si su token ha expirado.

```typescript
auth.guard.ts
```

```typescript
import { environment } from 'src/environments/environment';
import { LoginService } from './login.service';
import { Injectable } from '@angular/core';
import { CanActivate, RouterStateSnapshot, ActivatedRouteSnapshot, Router } from '@angular/router';
import { MenuService } from './menu.service';
import { JwtHelperService } from '@auth0/angular-jwt';
import * as CryptoJS from 'crypto-js';
import { Menu } from 'src/app/models/menu';
import { map } from 'rxjs/operators';

@Injectable({
  providedIn: 'root'
})
export class AuthGuard implements CanActivate {

  constructor(private router: Router, private loginService : LoginService, private menuservice: MenuService) { }
  
  canActivate(route: ActivatedRouteSnapshot, state: RouterStateSnapshot) {
    //logica que devuelva true or false   
    const helper = new JwtHelperService();

    let rpta = this.loginService.isLogged();

    if (!rpta) {
        localStorage.clear()
        this.router.navigate(['/login']);

        return false;
    } else {
        let password = "15646^&amp;%$3(),>2134bgGz*-+e7hds";

        let usuario = CryptoJS.AES.decrypt(localStorage.getItem('usuario'), password.trim()).toString(CryptoJS.enc.Utf8)
        let token = CryptoJS.AES.decrypt(localStorage.getItem('token'), password.trim()).toString(CryptoJS.enc.Utf8)

        let user = JSON.parse(usuario);

        if (!helper.isTokenExpired(token)) {
            let url = state.url;
            
          return this.menuservice.menuPorUsuario(user.cod_usuario, user.id_rzn_scl , token).pipe(map( (data : Menu[] )=>{
                this.menuservice.menuCambio.next(data);
                let cont = 0;
                for (let menuBD of data) {
                    if (url.startsWith(menuBD.txt_URL_OPC)) {
                      cont++;
                      break;
                    }
                  }
                if (cont > 0) {
                    return true;
                  } else {

                      if(url.startsWith('/dashboard')){

                        return true;
                      } else {

                        this.router.navigate(['others/access-denied']);
                        return false;
                      }
                  }
            }) );

        } else {
            localStorage.clear()
            this.router.navigate(['/login']);

            return false;
        }
    }
  } 
}
```
Para poder asegurar una ruta o modulo se tiene que agregar el GUARDS en el routing principal o en el routing de cada modulo individualmente.

```typescript
const adminRoutes: Routes = [
  { path: 'dashboard', component: DashboardComponent ,canActivate: [AuthGuard] },
  { path: 'notifications', component: NotificationsComponent, canActivate: [AuthGuard] },
 ];
```

O dentro de un modulo.


```typescript
{
    path: '',
    component: OthersLayoutComponent,
    canActivate : [OthersGuard],
    children: [
      {
        path: 'others',
        loadChildren: () => import('./pages/others/others.module').then(m => m.OthersModule)
      }
    ]
  }
```





