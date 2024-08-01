### 参数 skip test 区别

在构建Maven项目时，使用 `-DskipTests` 和 `-Dmaven.test.skip=true` 都可以达到跳过测试的目的，但在实际操作中这两者有一些细微的不同之处：

###### `-DskipTests`
当使用 `-DskipTests` 或 `-DskipTests=true` 时，Maven会跳过所有的测试生命周期阶段。  
在执行 `mvn clean install -DskipTests` 时，不会执行任何与测试相关的生命周期目标，例如 `test`, `verify`, `integration-test` 等。

###### `-Dmaven.test.skip=true`
使用 `-Dmaven.test.skip=true` 时，仅会跳过 `test` 生命周期阶段，即跳过单元测试。  
如果项目的构建过程依赖于测试之后的生命周期阶段（如 `verify`），这些阶段仍然会被执行。因此，`-Dmaven.test.skip=true` 只影响 `test` 阶段本身，而不影响后续阶段。

##### 具体区别

1. **生命周期阶段的影响**：
   - `-DskipTests` 会导致所有与测试相关的生命周期阶段都被跳过。
   - `-Dmaven.test.skip=true` 仅跳过 `test` 阶段，而后续阶段（如 `verify`）仍会被执行。
2. **默认值**：
   - `-DskipTests` 没有默认值。如果只使用 `-DskipTests` 而不指定 `true` 或 `false`，默认为 `true`。
   - `-Dmaven.test.skip` 的默认值为 `false`。如果不指定 `true` 或 `false`，默认不会跳过测试。
3. **可读性和规范性**：
   - `-Dmaven.test.skip=true` 更符合Maven的命名规范，更容易理解。

