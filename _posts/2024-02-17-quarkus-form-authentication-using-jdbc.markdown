---
layout: post
title: "Quarkus Form Authentication using JDBC"
date: 2024-02-17
tags: quarkus security java authentication database
---

While experimenting with [Quarkus](https://quarkus.io/) and its amazing
[Quinoa](https://quarkus.io/extensions/io.quarkiverse.quinoa/quarkus-quinoa/) extension,
I wanted to add form authentication to my project.
Quarkus offers a `BcryptUtil` utility class for hashing and verifying passwords. It also
provides two database authentication extensions. One for JPA and one using plain JDBC.
Since I wanted to use [jOOQ](https://www.jooq.org/) for persistence, the JPA extension
would add unnecessary dependencies to my project so JDBC was the clear choice.
Unfortunately the JDBC extension doesn't play along with the hashing utility which
expects passwords to be in the Modular Crypt Format (MCF). That's why I decided to
do some research and implement the authentication code myself using the built-in hashing
utility.

Here's how to get form authentication working using plain JDBC.

Let's first get the config out of the way:

```properties
quarkus.http.auth.form.enabled=true
quarkus.http.auth.form.username-parameter=email
quarkus.http.auth.form.password-parameter=password
quarkus.http.auth.form.post-location=/login

# SPA related configuration:
# do not redirect, respond with HTTP 200 OK
quarkus.http.auth.form.landing-page=
# do not redirect, respond with HTTP 401 Unauthorized
quarkus.http.auth.form.login-page=
quarkus.http.auth.form.error-page=
# HttpOnly must be false if you want to log out on the client; it can be true if logging out from the server
quarkus.http.auth.form.http-only-cookie=true
quarkus.http.auth.session.encryption-key=06c22ca5-fb87-440e-a6ed-67da77da69f5
```

Now form authentication is enabled on the `/login` endpoint using form inputs
named `email` and `password`.

Let's set up our database with a user account next:

```sql
create table account (
  id bigserial primary key,
  email text unique,
  password text,
  role text
);

insert into
account (email, password, role)
values ('admin@example.com', '$2a$10$BaPFwpG14G3Tn.avCOnOPuAQH5CvhS9a4XkvspoJESjoWxNdv2ryi', 'admin');
```

The password in this case is 'password'. Passwords can be created using the built-in `BcryptUtil` class.

Now we need to provide two `IdentityProvider` implementations for the form authentication
mechanism to be able to talk to the database. One is used for the initial login
whereas the second is used for refreshing the Cookie and populating the user roles.

```java
@ApplicationScoped
public class LoginIdentityProvider implements IdentityProvider<UsernamePasswordAuthenticationRequest> {

    private final DataSource dataSource;

    public LoginIdentityProvider(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    @Override
    public Class<UsernamePasswordAuthenticationRequest> getRequestType() {
        return UsernamePasswordAuthenticationRequest.class;
    }

    @Override
    public Uni<SecurityIdentity> authenticate(UsernamePasswordAuthenticationRequest request,
                                              AuthenticationRequestContext context) {

        return context.runBlocking(() -> {
            try (
                    Connection connection = dataSource.getConnection();
                    PreparedStatement statement = connection.prepareStatement("select password, role from account where email = ?");

            ) {
                statement.setString(1, request.getUsername());
                ResultSet resultSet = statement.executeQuery();
                if (!resultSet.next()) {
                    throw new AuthenticationFailedException();
                }

                String password = resultSet.getString("password");
                String role = resultSet.getString("role");
                String plainTextPassword = new String(request.getPassword().getPassword());

                if (!BcryptUtil.matches(plainTextPassword, password)) {
                    throw new AuthenticationFailedException();
                }

                return QuarkusSecurityIdentity.builder()
                        .setPrincipal(new QuarkusPrincipal(request.getUsername()))
                        .addCredential(request.getPassword())
                        .setAnonymous(false)
                        .addRole(role)
                        .build();
            } catch (SQLException e) {
                throw new RuntimeException(e);
            }
        });
    }

    @Override
    public int priority() {
        return 1001;
    }
}
```

The priority must be higher than 1000, so that it overrides any internal Quarkus
identity providers. We use `AuthenticationRequestContext#runBlocking` because of the
blocking nature of JDBC.

```java
@ApplicationScoped
public class AuthenticatedIdentityProvider implements IdentityProvider<TrustedAuthenticationRequest> {

    private final DataSource dataSource;

    public AuthenticatedIdentityProvider(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    @Override
    public Class<TrustedAuthenticationRequest> getRequestType() {
        return TrustedAuthenticationRequest.class;
    }

    @Override
    public Uni<SecurityIdentity> authenticate(TrustedAuthenticationRequest request,
                                              AuthenticationRequestContext context) {

        return context.runBlocking(() -> {
            try (
                    Connection connection = dataSource.getConnection();
                    PreparedStatement statement = connection.prepareStatement("select role from account where email = ?");

            ) {
                statement.setString(1, request.getPrincipal());
                ResultSet resultSet = statement.executeQuery();
                if (!resultSet.next()) {
                    throw new AuthenticationFailedException();
                }

                String role = resultSet.getString("role");

                return QuarkusSecurityIdentity.builder()
                        .setPrincipal(new QuarkusPrincipal(request.getPrincipal()))
                        .setAnonymous(false)
                        .addRole(role)
                        .build();
            } catch (SQLException e) {
                throw new RuntimeException(e);
            }
        });
    }

    @Override
    public int priority() {
        return 1001;
    }
}
```

That's all we need! If you're using jOOQ, you can also inject a `DSLContext` to run the queries.
